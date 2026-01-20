<!-- CITATION: This is based on this example: https://d3-graph-gallery.com/graph/heatmap_style.html -->
<script lang="ts">
	import data from './data.json';
	import * as d3 from 'd3';

	const margin = { top: 80, right: 25, bottom: 30, left: 100 };
	const width = 900 - margin.left - margin.right;
	const height = 450 - margin.top - margin.bottom;

	type RawDataPoint = {
		'Time to complete exam': number;
		Year: number;
		Nachklausur: string;
		Grade: number;
		Course: string;
		Attemptnumber: number;
		'Bachelor/Master': 'Bachelor' | 'Master';
		Study: string;
	};

	type DataPoint = {
		timeToComplete: number;
		year: number;
		resit: boolean;
		grade: number;
		course: string;
		attemptNumber: number;
		degree: string;
		study: string;
	};

	const filterState = $state({
		selectedCourse: 'ALL',
		resitValue: 'ALL',
		yearMin: 2021,
		yearMax: 2025
	});

	function mapData(data: RawDataPoint[]): DataPoint[] {
		return data.map(mapSingle);
	}

	function mapSingle(data: RawDataPoint): DataPoint {
		return {
			timeToComplete: data['Time to complete exam'],
			year: data['Year'],
			resit: data['Nachklausur'] === 'Yes',
			grade: data['Grade'],
			course: data['Course'],
			attemptNumber: data['Attemptnumber'],
			degree: data['Bachelor/Master'],
			study: data['Study']
		};
	}

	const mappedData: DataPoint[] = mapData(data as RawDataPoint[]);
	const filteredData = $derived(() => {
		return mappedData.filter((d) => {
			const courseMatch =
				filterState.selectedCourse === 'ALL' || d.course === filterState.selectedCourse;
			const resitMatch =
				filterState.resitValue === 'ALL' ||
				(filterState.resitValue === 'YES' && d.resit) ||
				(filterState.resitValue === 'NO' && !d.resit);
			const yearMatch = d.year >= filterState.yearMin && d.year <= filterState.yearMax;
			return courseMatch && resitMatch && yearMatch;
		});
	});
	const isError = $derived(() => filterState.yearMin > filterState.yearMax);

	const tiles = $derived(() => {
		const tileMap = new Map<string, DataPoint[]>();
		for (const d of filteredData()) {
			const x = binMinutes(d.timeToComplete, 10);
			const y = d.attemptNumber;
			const key = `${x}|${y}`;
			if (!tileMap.has(key)) {
				tileMap.set(key, []);
			}
			tileMap.get(key)!.push(d);
		}
		const tilesArray: { x: number; y: number; avgGrade: number; count: number }[] = [];
		for (const [key, points] of tileMap.entries()) {
			const [xStr, yStr] = key.split('|');
			const x = parseInt(xStr);
			const y = parseInt(yStr);
			const avgGrade = points.reduce((sum, p) => sum + p.grade, 0) / points.length;
			tilesArray.push({ x, y, avgGrade, count: points.length });
		}

		// fill rest with 0 count tiles
		const xValues = Array.from(
			new Set(filteredData().map((d) => binMinutes(d.timeToComplete, 10)))
		);
		const yValues = Array.from(new Set(filteredData().map((d) => d.attemptNumber)));
		for (const x of xValues) {
			for (const y of yValues) {
				const key = `${x}|${y}`;
				if (!tileMap.has(key)) {
					tilesArray.push({ x, y, avgGrade: 0, count: 0 });
				}
			}
		}
		return tilesArray;
	});

	function clearFilters() {
		filterState.selectedCourse = 'ALL';
		filterState.resitValue = 'ALL';
		filterState.yearMin = Math.min(...mappedData.map((d) => d.year));
		filterState.yearMax = Math.max(...mappedData.map((d) => d.year));
	}

	const gradeRange = [
		Math.min(...mappedData.map((d) => d.grade)),
		Math.max(...mappedData.map((d) => d.grade))
	];

	function binMinutes(min: number, binSize: number): number {
		return Math.floor(min / binSize) * binSize;
	}

	const x = d3
		.scaleBand()
		.range([0, width])
		.padding(0.05)
		.domain(
			Array.from(new Set(mappedData.map((d) => binMinutes(d.timeToComplete, 10))))
				.sort((a, b) => a - b)
				.map(String)
		);

	const y = d3
		.scaleBand()
		.range([height, 0])
		.padding(0.05)
		.domain(
			Array.from(new Set(mappedData.map((d) => d.attemptNumber)))
				.sort((a, b) => a - b)
				.map(String)
		);

	let xAxisG: SVGGElement | null = null;
	let yAxisG: SVGGElement | null = null;

	$effect(() => {
		if (xAxisG) {
			d3.select(xAxisG)
				.call(
					d3
						.axisBottom(x)
						.tickFormat((d) => `${parseInt(d)-10} - ${d} min`)
						.tickSize(0)
				)
				.select('.domain')
				.remove();
		}
		if (yAxisG) {
			d3.select(yAxisG)
				.call(
					d3
						.axisLeft(y)
						.tickFormat((d) => `Attempt ${d}`)
						.tickSize(0)
				)
				.select('.domain')
				.remove();
		}
	});

	const colorScale = d3
		.scaleSequential()
		.domain([gradeRange[0], gradeRange[1]])
		.interpolator(d3.interpolateBlues);

	let tooltip: HTMLDivElement | null = null;

	const handleMouseOver = (event: MouseEvent, d: DataPoint) => {
		if (!tooltip) return;
		tooltip.style = `position: absolute; left: ${event.pageX + 10}px; top: ${event.pageY + 10}px; background: white; border: 1px solid black; padding: 8px; border-radius: 4px; pointer-events: none;`;
		tooltip.innerHTML = `
			<strong>Average Grade:</strong> ${d.grade.toFixed(2)}<br/>
			<strong>Count of Students:</strong> ${
				filteredData().filter(
					(dp) =>
						binMinutes(dp.timeToComplete, 10) === binMinutes(d.timeToComplete, 10) &&
						dp.attemptNumber === d.attemptNumber
				).length
			}
		`;
	};

	const handleMouseMove = (event: MouseEvent) => {
		if (!tooltip) return;
		tooltip.style.left = `${event.pageX + 10}px`;
		tooltip.style.top = `${event.pageY + 10}px`;
	};

	const handleMouseOut = () => {
		if (!tooltip) return;
		tooltip.style.display = 'none';
	};
</script>

<div
	id="container"
	style={`margin-left: ${margin.left}px; margin-right: ${margin.right}px; width: ${width + 100}px;`}
>
	<div class="filter-controls">
		<label for="courseSelect">
			Course:
			<select name="courseSelect" id="courseSelect" bind:value={filterState.selectedCourse}>
				<option value="ALL">All Courses</option>
				{#each Array.from(new Set(mappedData.map((d) => d.course))).sort() as course}
					<option value={course}>
						{course}
					</option>
				{/each}
			</select>
		</label>
		<label for="resitSelect">
			Resit:
			<select name="resitSelect" id="resitSelect" bind:value={filterState.resitValue}>
				<option value="ALL">All</option>
				<option value="NO">No Resit</option>
				<option value="YES">Resit Only</option>
			</select>
		</label>
		<label for="yearMinSelect">
			Year Min:
			<select name="yearMinSelect" id="yearMinSelect" bind:value={filterState.yearMin}>
				{#each Array.from(new Set(mappedData.map((d) => d.year))).sort((a, b) => a - b) as year}
					<option value={year}>{year}</option>
				{/each}
			</select>
		</label>
		<label for="yearMaxSelect">
			Year Max:
			<select name="yearMaxSelect" id="yearMaxSelect" bind:value={filterState.yearMax}>
				{#each Array.from(new Set(mappedData.map((d) => d.year))).sort((a, b) => a - b) as year}
					<option value={year}>{year}</option>
				{/each}
			</select>
		</label>
		<button onclick={clearFilters}>Clear</button>
	</div>

	{#if isError()}
		<div class="error-container">
			{#if filteredData().length === 0}
				No data available for the selected filters.
			{/if}
			{#if filterState.yearMin > filterState.yearMax}
				Warning: Year Min is greater than Year Max.
			{/if}
		</div>
	{/if}

	<svg width={width + margin.left + margin.right} height={height + margin.top + margin.bottom}>
		<g transform={`translate(${margin.left}, ${margin.top})`}>
			{#each tiles() as tile}
				{#if tile.count === 0}
					<!-- draw gray rect -->
					<rect
						x={x(String(tile.x))}
						y={y(String(tile.y))}
						width={x.bandwidth()}
						height={y.bandwidth()}
						fill="gray"
						stroke="white"
						stroke-width="1"
					/>
				{:else}
					<rect
						x={x(String(tile.x))}
						y={y(String(tile.y))}
						width={x.bandwidth()}
						height={y.bandwidth()}
						fill={colorScale(tile.avgGrade)}
						stroke="white"
						stroke-width="1"
						onmouseover={(event) =>
							handleMouseOver(event, {
								timeToComplete: tile.x,
								attemptNumber: tile.y,
								grade: tile.avgGrade,
								year: 0,
								resit: false,
								course: '',
								degree: '',
								study: ''
							})}
						onfocus={() => 0}
						onmousemove={handleMouseMove}
						onmouseout={handleMouseOut}
						onblur={handleMouseOut}
						role="button"
						tabindex="0"
					/>
				{/if}
			{/each}
			<g bind:this={xAxisG} style="font-size: 15;" transform={`translate(0, ${height})`}></g>
			<g bind:this={yAxisG} style="font-size: 15;"></g>
		</g>

		<text
			x={(width + margin.left + margin.right) / 2}
			y={margin.top / 2}
			text-anchor="middle"
			font-size="20"
			font-weight="bold"
		>
			Exam Performance by Time to Complete and Attempt Number
		</text>
		<text
			x={(width + margin.left + margin.right) / 2}
			y={margin.top / 2 + 25}
			text-anchor="middle"
			font-size="14"
			fill="gray"
		>
			Average Grade represented by Color Scale, Lighter = Better, Gray = No Data
		</text>
	</svg>
	<div>
		<svg {width} height={50} style="margin-left: {margin.left}px; margin-right: {margin.right}px;">
			<defs>
				<linearGradient id="colorGradient" x1="0%" y1="0%" x2="100%" y2="0%">
					{#each d3.range(0, 1.01, 0.01) as t}
						<stop
							offset={`${t * 100}%`}
							stop-color={colorScale(gradeRange[0] + t * (gradeRange[1] - gradeRange[0]))}
						/>
					{/each}
				</linearGradient>
			</defs>
			<rect x={0} y={10} {width} height={20} fill="url(#colorGradient)" />
			<text x={0} y={45} font-size="12" text-anchor="start">
				{gradeRange[0].toFixed(2)} (best)
			</text>
			<text x={width} y={45} font-size="12" text-anchor="end">
				{gradeRange[1].toFixed(2)} (worst)
			</text>
			<text x={width / 2} y={45} font-size="12" text-anchor="middle"> Average Grade </text>
		</svg>
	</div>

	<div bind:this={tooltip} class="tooltip" style="position: absolute; pointer-events: none;"></div>
</div>

<style>
	#container {
		font-family: Arial, sans-serif;
		font-size: 14px;
	}

	.filter-controls {
		display: flex;
		gap: 20px;
		margin-bottom: 20px;
		margin-top: 20px;
		margin-left: 60px;
	}

	.error-container {
		color: red;
		margin-bottom: 10px;
		margin-left: 60px;
		background-color: #ffe6e6;
		padding: 10px;
		border: 1px solid red;
		border-radius: 5px;
		width: 100%;
	}
</style>
