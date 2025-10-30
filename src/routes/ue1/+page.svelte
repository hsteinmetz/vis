<script lang="ts">
	import * as d3 from 'd3';

	import data from './Uebung01_Daten.json';

	const width = 800;
	const height = 600;
	const marginTop = 50;
	const marginRight = 50;
	const marginBottom = 50;
	const marginLeft = 50;

	let x = d3
		.scaleBand()
		.domain(data.map((d) => d.Stadt))
		.range([marginLeft, width - marginRight])
		.padding(0.1);
	let y = d3
		.scaleLinear()
		.domain([0, d3.max(data, (d) => d.Anzahl_Kebabläden) ?? 0])
		.range([height - marginBottom, marginTop]);

	let gx: SVGGElement;
	let gy: SVGGElement;

	$effect(() => {
		d3.select(gx).call(d3.axisBottom(x));

		d3.select(gy).call(d3.axisLeft(y));
	});
</script>

<svg {width} {height} id="container">
	<g bind:this={gx} transform="translate(0, {height - marginBottom})" />
	<g bind:this={gy} transform="translate({marginLeft}, 0)" />
	<g fill="steelblue">
		{#each data as d}
			<rect
				x={x(d.Stadt)}
				y={y(d.Anzahl_Kebabläden)}
				height={y(0) - y(d.Anzahl_Kebabläden)}
				width={x.bandwidth()}
			/>
		{/each}
	</g>
</svg>
