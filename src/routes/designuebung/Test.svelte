<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { scaleBand, scaleSequential } from 'd3-scale';
	import { extent } from 'd3-array';
	import { interpolateCividis, interpolateViridis } from 'd3-scale-chromatic';
	import { axisBottom, axisLeft } from 'd3-axis';
	import { select } from 'd3-selection';

	type Row = {
		timeToComplete: number;
		year: number;
		resit: boolean;
		grade: number;
		course: string;
		attemptNumber: number;
		degree: string;
		study: string;
	};

	type HeatmapCell = {
		x: number; // bin start
		y: number; // attempt
		count: number;
		avgGrade: number;
	};

	export let data: Row[] = [];

	// --- UI state ---
	let selectedCourse: string = 'ALL';
	let resitValue: 'ALL' | 'YES' | 'NO' = 'ALL';
	let yearMin = 0;
	let yearMax = 0;

	let binSize: 5 | 10 | 15 | 20 = 10;

	// --- Derived filter options ---
	$: courses = Array.from(new Set(data.map((d) => d.course))).sort();
	$: years = Array.from(new Set(data.map((d) => d.year))).sort((a, b) => a - b);

	$: if (years.length && yearMin === 0 && yearMax === 0) {
		yearMin = years[0];
		yearMax = years[years.length - 1];
	}

	// --- Filtered rows ---
	$: filtered = data.filter((d) => {
		if (selectedCourse !== 'ALL' && d.course !== selectedCourse) return false;
		if (resitValue === 'YES' && d.resit !== true) return false;
		if (resitValue === 'NO' && d.resit !== false) return false;
		if (d.year < yearMin || d.year > yearMax) return false;
		if (d.attemptNumber < 1 || d.attemptNumber > 3) return false;
		return true;
	});

	// --- Heatmap aggregation ---
	function binMinutes(min: number, size: number): number {
		return Math.floor(min / size) * size;
	}

	function buildCells(rows: Row[], size: number): HeatmapCell[] {
		const m = new Map<string, { x: number; y: number; count: number; sumGrade: number }>();

		for (const d of rows) {
			const x = binMinutes(d.timeToComplete, size);
			const y = d.attemptNumber;
			const key = `${x}|${y}`;

			const cell = m.get(key);
			if (!cell) {
				m.set(key, { x, y, count: 1, sumGrade: d.grade });
			} else {
				cell.count += 1;
				cell.sumGrade += d.grade;
			}
		}

		return Array.from(m.values()).map((c) => ({
			x: c.x,
			y: c.y,
			count: c.count,
			avgGrade: c.sumGrade / c.count
		}));
	}

	$: cells = buildCells(filtered, binSize);

	// --- x bins (ensure full grid incl. empty bins) ---
	$: minutesExt = filtered.length
		? (extent(filtered, (d) => d.timeToComplete) as [number, number])
		: [0, 0];

	$: xBins = (() => {
		const [minV, maxV] = minutesExt;
		if (!Number.isFinite(minV) || !Number.isFinite(maxV) || minV === maxV) return [] as number[];
		const start = Math.floor(minV / binSize) * binSize;
		const end = Math.ceil(maxV / binSize) * binSize;
		const bins: number[] = [];
		for (let b = start; b <= end; b += binSize) bins.push(b);
		return bins;
	})();

	const attempts: number[] = [1, 2, 3];

	// --- Layout / Responsive ---
	let container: HTMLDivElement | null = null;
	let ro: ResizeObserver | null = null;

	let width = 900;
	let height = 480;
	const margin = { top: 20, right: 20, bottom: 60, left: 60 };

	onMount(() => {
		ro = new ResizeObserver((entries) => {
			const w = entries[0]?.contentRect?.width;
			if (typeof w === 'number' && w > 400) width = w;
		});
		if (container) ro.observe(container);
	});

	onDestroy(() => {
		if (ro && container) ro.unobserve(container);
	});

	$: innerW = Math.max(10, width - margin.left - margin.right);
	$: innerH = Math.max(10, height - margin.top - margin.bottom);

	// --- Scales ---
	$: xScale = scaleBand<string>()
		.domain(xBins.map(String))
		.range([0, innerW])
		.paddingInner(0.05)
		.paddingOuter(0.02);

	$: yScale = scaleBand<string>()
		.domain(attempts.map(String))
		.range([0, innerH])
		.paddingInner(0.1)
		.paddingOuter(0.05);

	// Color scale domain from avgGrade
	$: gradeExt = cells.length ? (extent(cells, (c) => c.avgGrade) as [number, number]) : [0, 1];

	// IMPORTANT: Wenn 1.0 gut und 6.0 schlecht ist, ist invertieren sinnvoll:
	$: colorScale = scaleSequential(interpolateViridis).domain([gradeExt[1], gradeExt[0]]);
	let pinned: HeatmapCell | null = null;
	function togglePin(cell: HeatmapCell): void {
		if (pinned && pinned.x === cell.x && pinned.y === cell.y) pinned = null;
		else pinned = cell;
	}

	// optional helper for styling pinned outline
	function isPinned(cell: HeatmapCell): boolean {
		return !!pinned && pinned.x === cell.x && pinned.y === cell.y;
	}

	// --- Axes ---
	let xAxisG: SVGGElement | null = null;
	let yAxisG: SVGGElement | null = null;

	function renderAxes(): void {
		if (!xAxisG || !yAxisG) return;

		// ticks ausdünnen
		const step = Math.max(1, Math.ceil(40 / binSize));
		const tickValues = xBins.filter((_, i) => i % step === 0).map(String);

		select(xAxisG)
			.call(
				axisBottom(xScale)
					.tickValues(tickValues)
					.tickFormat((d: any) => `${d}-${Number(d) + binSize}`)
			)
			.call((g) =>
				g.selectAll('text').attr('transform', 'rotate(-35)').style('text-anchor', 'end')
			);

		select(yAxisG).call(axisLeft(yScale).tickFormat((d: any) => `Attempt ${d}`));
	}

	$: if (xScale && yScale) {
		queueMicrotask(renderAxes);
	}

	// --- Tooltip ---
	type TooltipState = { show: boolean; x: number; y: number; content: string };
	let tooltip: TooltipState = { show: false, x: 0, y: 0, content: '' };

	function onCellMove(event: MouseEvent, cell: HeatmapCell): void {
		const pad = 12;
		tooltip = {
			show: true,
			x: event.clientX + pad,
			y: event.clientY + pad,
			content:
				`Minutes: ${cell.x}-${cell.x + binSize}\n` +
				`Attempt: ${cell.y}\n` +
				`Avg grade: ${cell.avgGrade.toFixed(2)}\n` +
				`Count: ${cell.count}`
		};
	}

	function hideTooltip(): void {
		tooltip = { ...tooltip, show: false };
	}

	$: cellMap = (() => {
		const m = new Map<string, HeatmapCell>();
		for (const c of cells) m.set(`${c.x}|${c.y}`, c);
		return m;
	})();

	function getCell(xBin: number, attempt: number): HeatmapCell | undefined {
		return cellMap.get(`${xBin}|${attempt}`);
	}
</script>

<div bind:this={container} style="width: 100%;">
	<div style="display:flex; gap:12px; flex-wrap:wrap; align-items:end; margin-bottom:10px;">
		<label>
			Course
			<select bind:value={selectedCourse}>
				<option value="ALL">All</option>
				{#each courses as c}
					<option value={c}>{c}</option>
				{/each}
			</select>
		</label>

		<label>
			Nachklausur
			<select bind:value={resitValue}>
				<option value="ALL">All</option>
				<option value="YES">Yes</option>
				<option value="NO">No</option>
			</select>
		</label>

		{#if years.length}
			<label>
				Year min
				<select bind:value={yearMin}>
					{#each years as y}
						<option value={y}>{y}</option>
					{/each}
				</select>
			</label>

			<label>
				Year max
				<select bind:value={yearMax}>
					{#each years as y}
						<option value={y}>{y}</option>
					{/each}
				</select>
			</label>
		{/if}

		<label>
			Bin size (minutes)
			<select bind:value={binSize}>
				<option value={5}>5</option>
				<option value={10}>10</option>
				<option value={15}>15</option>
				<option value={20}>20</option>
			</select>
		</label>

		<div style="margin-left:auto; font-size: 0.9em;">
			Rows: {filtered.length} | Cells: {cells.length}
		</div>
	</div>

	<!-- Legend + Pinned summary (compact) -->

	<div style="display:flex; gap:16px; align-items:center; flex-wrap:wrap; margin-bottom:10px;">
		<!-- Color legend -->

		<div style="display:flex; align-items:center; gap:10px;">
			<div style="font-size:12px; min-width: 80px;">Avg grade</div>
			<svg width="220" height="28" aria-label="legend">
				<defs>
					<linearGradient id="gradeLegend" x1="0" x2="1" y1="0" y2="0">
						<!-- domain is inverted for 1.0 best -> show best on the left -->
						<stop offset="0%" stop-color={colorScale(gradeExt[0])} />
						<stop offset="100%" stop-color={colorScale(gradeExt[1])} />
					</linearGradient>
				</defs>
				<rect x="0" y="6" width="160" height="12" fill="url(#gradeLegend)" stroke="#ccc" />
				<text x="0" y="26" font-size="11">{gradeExt[0].toFixed(2)} (best)</text>

				<text x="160" y="26" font-size="11" text-anchor="end">{gradeExt[1].toFixed(2)} (worst)</text
				>
			</svg>
		</div>

		<div style="font-size:12px;">
			{#if pinned}
				<div><strong>Pinned</strong></div>

				<div>Minutes: {pinned.x}-{pinned.x + binSize}</div>

				<div>Attempt: {pinned.y}</div>

				<div>Avg grade: {pinned.avgGrade.toFixed(2)}</div>

				<div>Count: {pinned.count}</div>
				<button type="button" on:click={() => (pinned = null)} style="margin-top:6px;">
					Clear
				</button>
			{:else}
				<div><strong>Pinned</strong>: click a colored cell</div>
			{/if}
		</div>
	</div>

	<svg {width} {height} style="border:1px solid #ddd; background:white;">
		<g transform={`translate(${margin.left},${margin.top})`}>
			{#if xBins.length}
				{#each attempts as a}
					{#each xBins as xb}
						{#key `${xb}|${a}`}
							{#if getCell(xb, a)}
								<rect
									x={xScale(String(xb))}
									y={yScale(String(a))}
									width={xScale.bandwidth()}
									height={yScale.bandwidth()}
									fill={colorScale(getCell(xb, a)!.avgGrade)}
									opacity={0.95}
									on:mousemove={(e) => onCellMove(e as unknown as MouseEvent, getCell(xb, a)!)}
									on:click={() => togglePin(getCell(xb, a)!)}
									stroke={isPinned(getCell(xb, a)!) ? '#000' : 'none'}
									stroke-width={isPinned(getCell(xb, a)!) ? 2 : 0}
									on:mouseleave={hideTooltip}
								/>
							{:else}
								<rect
									x={xScale(String(xb))}
									y={yScale(String(a))}
									width={xScale.bandwidth()}
									height={yScale.bandwidth()}
									fill="#f5f5f5"
								/>
							{/if}
						{/key}
					{/each}
				{/each}
			{/if}

			<g bind:this={xAxisG} transform={`translate(0,${innerH})`} />
			<g bind:this={yAxisG} />

			<text x={innerW / 2} y={innerH + 52} text-anchor="middle" font-size="12">
				Minutes to complete (binned)
			</text>
			<text x={-innerH / 2} y={-45} transform="rotate(-90)" text-anchor="middle" font-size="12">
				Attempt
			</text>
		</g>
	</svg>
</div>

{#if tooltip.show}
	<div
		style="
      position: fixed;
      left: {tooltip.x}px;
      top: {tooltip.y}px;
      padding: 8px 10px;
      background: rgba(0,0,0,0.85);
      color: white;
      font-size: 12px;
      border-radius: 6px;
      pointer-events: none;
      white-space: pre;
      z-index: 9999;
      max-width: 280px;"
	>
		{tooltip.content}
	</div>
{/if}
