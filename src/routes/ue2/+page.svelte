<script lang="ts">
	import data from './Uebung02_Daten.json';
	import * as d3 from 'd3';

	const width = 800;
	const height = 600;
	const marginTop = 50;
	const marginRight = 50;
	const marginBottom = 50;
	const marginLeft = 50;

	let x = d3
		.scaleBand()
		.domain(data.map((d) => d.Jahr))
		.range([marginLeft, width - marginRight]);
	let y = d3
		.scaleLinear()
		.domain([0, d3.max(data, (d) => d['Anzahl Kebabläden']) ?? 0])
		.range([height - marginBottom, marginTop]);

	let gx: SVGGElement;
	let gy: SVGGElement;

	let line = d3.line(
		(d) => x(d.Jahr)! + x.bandwidth() / 2,
		(d) => y(d['Anzahl Kebabläden'])
	);

	let gridLine = d3.line(
		(d) => x(d),
		(d) => y(d)
	);

	let data_koeln = $derived(data.filter((d) => d.Stadt === 'Köln'));
	let data_berlin = $derived(data.filter((d) => d.Stadt === 'Berlin'));

	$effect(() => {
		d3.select(gx).call(d3.axisBottom(x));
		d3.select(gy).call(d3.axisLeft(y));
	});
</script>

<svg {width} {height}>
	<g bind:this={gx} transform="translate(0, {height - marginBottom})" />
	<g bind:this={gy} transform="translate({marginLeft}, 0)">
		{#each y.ticks().filter((tick) => tick % 200 === 0) as tick}
			<line
				x1={0}
				x2={width - 2 * marginRight}
				y1={y(tick)}
				y2={y(tick)}
				stroke="grey"
				stroke-opacity="0.4"
			/>
		{/each}
	</g>

	<path d={line(data_koeln)} fill="none" stroke="steelblue" stroke-width="2" />
	<path d={line(data_berlin)} fill="none" stroke="steelblue" stroke-width="2" />
</svg>
