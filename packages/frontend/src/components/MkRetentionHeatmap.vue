<template>
<div ref="rootEl">
	<MkLoading v-if="fetching"/>
	<div v-else>
		<canvas ref="chartEl"></canvas>
	</div>
</div>
</template>

<script lang="ts" setup>
import { markRaw, version as vueVersion, onMounted, onBeforeUnmount, nextTick } from 'vue';
import {
	Chart,
	ArcElement,
	LineElement,
	BarElement,
	PointElement,
	BarController,
	LineController,
	CategoryScale,
	LinearScale,
	TimeScale,
	Legend,
	Title,
	Tooltip,
	SubTitle,
	Filler,
} from 'chart.js';
import { enUS } from 'date-fns/locale';
import tinycolor from 'tinycolor2';
import * as os from '@/os';
import 'chartjs-adapter-date-fns';
import { defaultStore } from '@/store';
import { useChartTooltip } from '@/scripts/use-chart-tooltip';
import { MatrixController, MatrixElement } from 'chartjs-chart-matrix';
import { chartVLine } from '@/scripts/chart-vline';
import { alpha } from '@/scripts/color';

Chart.register(
	ArcElement,
	LineElement,
	BarElement,
	PointElement,
	BarController,
	LineController,
	CategoryScale,
	LinearScale,
	TimeScale,
	Legend,
	Title,
	Tooltip,
	SubTitle,
	Filler,
	MatrixController, MatrixElement,
);

const rootEl = $ref<HTMLDivElement>(null);
const chartEl = $ref<HTMLCanvasElement>(null);
const now = new Date();
let chartInstance: Chart = null;
let fetching = $ref(true);

const { handler: externalTooltipHandler } = useChartTooltip({
	position: 'middle',
});

async function renderChart() {
	if (chartInstance) {
		chartInstance.destroy();
	}

	const wide = rootEl.offsetWidth > 600;
	const narrow = rootEl.offsetWidth < 400;

	const maxDays = wide ? 20 : narrow ? 10 : 15;

	const raw = await os.api('retention', { });

	const data = [];
	for (const record of raw) {
		let i = 0;
		for (const date of Object.keys(record.data).sort((a, b) => new Date(a).getTime() - new Date(b).getTime())) {
			data.push({
				x: i,
				y: record.createdAt,
				v: record.data[date],
			});
			i++;
		}
	}

	console.log(data);

	fetching = false;

	await nextTick();

	const gridColor = defaultStore.state.darkMode ? 'rgba(255, 255, 255, 0.1)' : 'rgba(0, 0, 0, 0.1)';

	// フォントカラー
	Chart.defaults.color = getComputedStyle(document.documentElement).getPropertyValue('--fg');

	const color = defaultStore.state.darkMode ? '#b4e900' : '#86b300';

	// 視覚上の分かりやすさのため上から最も大きい3つの値の平均を最大値とする
	//const max = raw.readWrite.slice().sort((a, b) => b - a).slice(0, 3).reduce((a, b) => a + b, 0) / 3;
	const max = 4;

	const marginEachCell = 6;

	chartInstance = new Chart(chartEl, {
		type: 'matrix',
		data: {
			datasets: [{
				label: 'Active',
				data: data,
				pointRadius: 0,
				borderWidth: 0,
				borderJoinStyle: 'round',
				borderRadius: 3,
				backgroundColor(c) {
					const value = c.dataset.data[c.dataIndex].v;
					const a = value / max;
					return alpha(color, a);
				},
				fill: true,
				width(c) {
					const a = c.chart.chartArea ?? {};
					return (a.right - a.left) / maxDays - marginEachCell;
				},
				height(c) {
					const a = c.chart.chartArea ?? {};
					return (a.bottom - a.top) / maxDays - (marginEachCell / 1.5);
				},
			}],
		},
		options: {
			aspectRatio: wide ? 2 : narrow ? 2 : 2,
			layout: {
				padding: {
					left: 8,
					right: 0,
					top: 0,
					bottom: 0,
				},
			},
			scales: {
				x: {
					position: 'top',
					suggestedMax: maxDays,
					grid: {
						display: false,
						color: gridColor,
						borderColor: 'rgb(0, 0, 0, 0)',
					},
					ticks: {
						display: true,
						maxRotation: 0,
						autoSkipPadding: 8,
						callback: (value, index, values) => value + 1,
					},
				},
				y: {
					type: 'time',
					min: new Date(new Date().getFullYear(), new Date().getMonth(), new Date().getDate() - maxDays),
					offset: true,
					reverse: true,
					position: 'left',
					time: {
						unit: 'day',
						round: 'day',
					},
					grid: {
						display: false,
						color: gridColor,
						borderColor: 'rgb(0, 0, 0, 0)',
					},
					ticks: {
						maxRotation: 0,
						autoSkip: true,
						padding: 1,
						font: {
							size: 9,
						},
					},
				},
			},
			animation: false,
			plugins: {
				legend: {
					display: false,
				},
				tooltip: {
					enabled: false,
					callbacks: {
						title(context) {
							const v = context[0].dataset.data[context[0].dataIndex];
							return v.d;
						},
						label(context) {
							const v = context.dataset.data[context.dataIndex];
							return ['Active: ' + v.v];
						},
					},
					//mode: 'index',
					animation: {
						duration: 0,
					},
					external: externalTooltipHandler,
				},
			},
		},
	});
}

onMounted(async () => {
	renderChart();
});
</script>