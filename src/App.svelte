<script lang="ts">
	import {
		onMount
	} from 'svelte';
	import * as d3 from 'd3';
	import { crossfade } from 'svelte/transition';
	import { flip } from 'svelte/animate';

	const {log} = console;

	let numToShow = 20;

	let data = [];
	let colorAccent = d3.scaleOrdinal(d3.schemeTableau10);
	const [send, receive] = crossfade({
		duration: d => 10,

		fallback(node, params) {
			const style = getComputedStyle(node);
			const transform = style.transform === 'none' ? '' : style.transform;

			return {
				duration: 10,
				css: t => `
					transform: ${transform};
				`
			};
		}
	});
	onMount(async () => {
		data = await d3.csv('https://raw.githubusercontent.com/CSSEGISandData/COVID-19/master/csse_covid_19_data/csse_covid_19_time_series/time_series_covid19_confirmed_global.csv', (d) => {
			// log(d);
			Object.entries(d).forEach(([key, value]) => {
				if (/^(?:\d{1,2}\/){2}\d{1,2}$/.test(key)) {
					const [month, day, year] = key.split('/').map(v => +v);
					const date = new Date(2000 + year, month - 1, day);
					const dateStr = `${date.getFullYear()}/${date.getMonth() + 1}/${date.getDate()}`;
					delete d[key];
					d[dateStr] = +value;
				}
			});
			return d;
		});
		log({data});
	});
	
	let currentYearIndex = 0;
	let totalDataToShow = [];
	$: dataToShow = totalDataToShow[currentYearIndex] || {};
	$: if (data.length > 0) {
		let groupedByDate: object = data.reduce((result, country, index) => {
			for (const prop in country) {
				if (country.hasOwnProperty(prop) && /^\d{4}(?:\/\d{1,2}){2}$/.test(prop)) {
					if (result[prop] === undefined) {
						result[prop] = [{
							name: country['Country/Region'],
							value: country[prop],
							color: colorAccent(index)
						}];
					} else {
						let currCountry = result[prop].find(({name}) => name === country['Country/Region']);
						if (currCountry === undefined) {
							result[prop].push({
								name: country['Country/Region'],
								value: country[prop],
								color: colorAccent(index)
							});
						} else { 
							currCountry.value += country[prop];
						}
					}
				}
			}
			return result;
		}, {});
		
		const datesArray = Object.entries(groupedByDate)
			.map(([key, value]) => ({
				date: key,
				countries: value
			}));
		datesArray.forEach(({countries}) => {
			countries.sort((a, b) => b.value - a.value);
		});		
		totalDataToShow = datesArray;
		/**
		 * {
		 * 		date: '',
		 * 		countries: 
		 * }
		*/
	}

	let lastUpdatedTime = 0;
	const updateChart = (timing) => {
		// log(timing, lastUpdatedTime, timing - lastUpdatedTime);
		if (lastUpdatedTime === 0) lastUpdatedTime = timing;
		if (currentYearIndex < totalDataToShow.length - 1) {
			if (timing - lastUpdatedTime > 500) {
			currentYearIndex++;
			lastUpdatedTime = timing;
		}
			requestAnimationFrame(updateChart);
		}
	};
	
	$: if (totalDataToShow.length > 0 && lastUpdatedTime === 0) {
		requestAnimationFrame(updateChart);
	}
	

	const getScale = (data = []) => {
		return d3.scaleLinear()
			.domain([data[data.length - 1].value, data[0].value])
			.range([10, 100]);
	};

	$: scale = dataToShow.countries && getScale(dataToShow.countries.slice(0, numToShow)) || (() => 100);
	$: barChartContainerStyle = `height: ${numToShow * 35 + numToShow * 8}px`;
	const getTranslateWidth = (scale, num) => {
		return scale(num);
	};

	const getBarStyle = (scale, country) => {
		return `${getTranslateXStyle(scale, country.value)}; background-color: ${country.color}`;
	}

	const getTranslateXStyle = (scale, num: number) => {
		return `transform: translateX(${getTranslateWidth(scale, num)}%)`;
	};

	
</script>

<main>
	<h1>Bar Chart Race of Covid cases {dataToShow.date}</h1>
</main>
<section>
	<div class="container">
		<div class="left-axis">
			{#each {length: numToShow} as _, i}
				<div class="sn">
					{i + 1}
				</div>
				
			{/each}
		</div>
		<div class="bar-chart-container" style={barChartContainerStyle}>
			{#if dataToShow.countries}
				{#each dataToShow.countries.slice(0, numToShow + 10) as country (country.name)}
					<div 
						in:receive="{{key: country.name}}"
						out:send="{{key: country.name}}"
						animate:flip
						class="bar-chart"
						style={getBarStyle(scale, country)}>
						<div class="detail-container">
							<div>{country.name}</div>
							<div>{country.value}</div>
						</div>
					</div>
				{/each}
			{/if}
		</div>
	</div>
	
</section>

<style>
	.container {
		display: flex;
		overflow: hidden;
		padding: 16px 0;
	}
	.left-axis {
		display: flex;
		flex-direction: column;
		align-items: flex-end;
	}
	.sn {
		display: flex;
		align-items: center;
		margin-top: 8px;
		min-height: 35px;
	}
	.bar-chart-container {
		flex: 1;
		position: relative;
		margin-left: 8px;
		overflow: hidden;
	}
	.bar-chart {
		border-radius: 5px;
		display: flex;
		justify-content: flex-end;
		margin-left: -100%;
		width: 100%;
		height: 35px;
		background-color: red;
		margin-top: 8px;
		transition: transform 0.5s
	}
	.detail-container {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: flex-end;
		margin-right: 8px;
		color: white;
		text-align: right;
		font-size: 12px;
	}

</style>