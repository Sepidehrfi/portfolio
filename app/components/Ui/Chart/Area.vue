<template>
	<div :class="styles({ class: props.class })">
		<UiChartLegend
			v-if="showLegend"
			v-model:items="legendItems"
			@legend-item-click="handleLegendItemClick"
		/>

		<VisXYContainer
			:style="{ height: isMounted ? '100%' : 'auto' }"
			:margin="{ left: 20, right: 20 }"
			:data="data"
		>
			<svg width="0" height="0">
				<defs>
					<linearGradient
						v-for="(color, i) in colors"
						:id="`${chartRef}-color-${i}`"
						:key="i"
						x1="0"
						y1="0"
						x2="0"
						y2="1"
					>
						<template v-if="showGradiant">
							<stop offset="5%" :stop-color="color" stop-opacity="0.4" />
							<stop offset="95%" :stop-color="color" stop-opacity="0" />
						</template>
						<template v-else>
							<stop offset="0%" :stop-color="color" />
						</template>
					</linearGradient>
				</defs>
			</svg>

			<UiChartCrosshair
				v-if="showTooltip"
				:colors="colors"
				:items="legendItems"
				:index="index"
				:custom-tooltip="customTooltip"
			/>

			<template v-for="(category, i) in categories" :key="category">
				<VisArea
					:x="(d: Data, i: number) => i"
					:y="(d: Data) => d[category]"
					color="auto"
					:curve-type="curveType"
					:attributes="{
						[Area.selectors.area]: {
							fill: `url(#${chartRef}-color-${i})`,
						},
					}"
					:opacity="
						legendItems.find((item) => item.name === category)?.inactive ? filterOpacity : 1
					"
				/>
			</template>

			<template v-for="(category, i) in categories" :key="category">
				<VisLine
					:x="(d: Data, i: number) => i"
					:y="(d: Data) => d[category]"
					:color="colors[i]"
					:curve-type="curveType"
					:attributes="{
						[Line.selectors.line]: {
							opacity: legendItems.find((item) => item.name === category)?.inactive
								? filterOpacity
								: 1,
						},
					}"
				/>
			</template>

			<VisAxis
				v-if="showXAxis"
				type="x"
				:tick-format="xFormatter ?? ((v: number) => data[v]?.[index])"
				:grid-line="false"
				:tick-line="false"
				tick-text-color="hsl(var(--vis-text-color))"
			/>
			<VisAxis
				v-if="showYAxis"
				type="y"
				:tick-line="false"
				:tick-format="yFormatter"
				:domain-line="false"
				:grid-line="showGridLine"
				:attributes="{
					[Axis.selectors.grid]: {
						class: 'text-muted',
					},
				}"
				tick-text-color="hsl(var(--vis-text-color))"
			/>

			<slot />
		</VisXYContainer>
	</div>
</template>

<script lang="ts">
	import { Area, Axis, CurveType, Line } from "@unovis/ts";
	import { VisArea, VisAxis, VisLine, VisXYContainer } from "@unovis/vue";
	import type { BulletLegendItemInterface, Spacing } from "@unovis/ts";

	import { defaultColors } from "./chart";

	type KeyOf<T extends Record<string, any>> = Extract<keyof T, string>;

	export interface AreaChartProps<T extends Record<string, any>> {
		/**
		 * The source data, in which each entry is a dictionary.
		 */
		data: T[];
		/**
		 * Select the categories from your data. Used to populate the legend and toolip.
		 */
		categories: KeyOf<T>[];
		/**
		 * Sets the key to map the data to the axis.
		 */
		index: KeyOf<T>;
		/**
		 * Change the default colors.
		 */
		colors?: string[];
		/**
		 * Margin of each the container
		 */
		margin?: Spacing;
		/**
		 * Change the opacity of the non-selected field
		 * @default 0.2
		 */
		filterOpacity?: number;
		/**
		 * Function to format X label
		 */
		xFormatter?: (tick: number | Date, i: number, ticks: number[] | Date[]) => string;
		/**
		 * Function to format Y label
		 */
		yFormatter?: (tick: number | Date, i: number, ticks: number[] | Date[]) => string;
		/**
		 * Controls the visibility of the X axis.
		 * @default true
		 */
		showXAxis?: boolean;
		/**
		 * Controls the visibility of the Y axis.
		 * @default true
		 */
		showYAxis?: boolean;
		/**
		 * Controls the visibility of tooltip.
		 * @default true
		 */
		showTooltip?: boolean;
		/**
		 * Controls the visibility of legend.
		 * @default true
		 */
		showLegend?: boolean;
		/**
		 * Controls the visibility of gridline.
		 * @default true
		 */
		showGridLine?: boolean;
	}
</script>

<script setup lang="ts" generic="T extends Record<string, any>">
	const styles = tv({
		base: "flex h-[400px] w-full flex-col items-end",
	});

	const props = withDefaults(
		defineProps<
			AreaChartProps<T> & {
				/**
				 * Render custom tooltip component.
				 */
				customTooltip?: Component;
				/**
				 * Type of curve
				 */
				curveType?: CurveType;
				/**
				 * Controls the visibility of gradient.
				 * @default true
				 */
				showGradiant?: boolean;
				/**
				 * Additional class to be added to the container.
				 */
				class?: any;
			}
		>(),
		{
			curveType: CurveType.MonotoneX,
			filterOpacity: 0.2,
			margin: () => ({ top: 0, bottom: 0, left: 0, right: 0 }),
			showXAxis: true,
			showYAxis: true,
			showTooltip: true,
			showLegend: true,
			showGridLine: true,
			showGradiant: true,
		}
	);

	const emits = defineEmits<{
		legendItemClick: [d: BulletLegendItemInterface, i: number];
	}>();

	type KeyOfT = Extract<keyof T, string>;
	type Data = (typeof props.data)[number];

	const chartRef = useId();

	const index = computed(() => props.index as KeyOfT);
	const colors = computed(() =>
		props.colors?.length ? props.colors : defaultColors(props.categories.length)
	);

	const legendItems = ref<BulletLegendItemInterface[]>(
		props.categories.map((category, i) => ({
			name: category,
			color: colors.value[i],
			inactive: false,
		}))
	);

	const isMounted = useMounted();

	function handleLegendItemClick(d: BulletLegendItemInterface, i: number) {
		emits("legendItemClick", d, i);
	}
</script>
