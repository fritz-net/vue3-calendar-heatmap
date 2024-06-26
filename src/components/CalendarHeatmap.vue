<template>
	<div :class="{'vch__container': true, 'dark-mode': darkMode}">
		<svg class="vch__wrapper" ref="svg" :viewBox="viewbox">
			<g class="vch__months__labels__wrapper" :transform="monthsLabelWrapperTransform">
				<text
					class="vch__month__label"
					v-for="(month, index) in heatmap.firstFullWeekOfMonths"
					:key="index"
					:x="getMonthLabelPosition(month).x"
					:y="getMonthLabelPosition(month).y"
				>
					{{ lo.months[ month.value ] }}
				</text>
			</g>
            
            <g class="vch__days__labels__wrapper" :transform="daysLabelWrapperTransform">
            <template v-for="i in [0, 1, 2, 3, 4, 5, 6]">
				<text class="vch__day__label" :key="i"
                      v-if="(i + startWeekday) % 2 == 1"
					  :x="vertical ? SQUARE_SIZE * i : 0"
					  :y="vertical ? SQUARE_SIZE - SQUARE_BORDER_SIZE : ((SQUARE_SIZE - SQUARE_BORDER_SIZE) + SQUARE_SIZE * i)"
				>
					{{ lo.days[ (i + startWeekday) % DAYS_IN_WEEK ] }}
				</text>
            </template>
            </g>

			<g v-if="vertical" class="vch__legend__wrapper" :transform="legendWrapperTransform">
				<text :x="SQUARE_SIZE * 1.25" y="8">{{ lo.less }}</text>
				<rect
					v-for="(color, index) in curRangeColor"
					:key="index"
					:rx="round"
					:ry="round"
					:style="{ fill: color }"
					:width="SQUARE_SIZE - SQUARE_BORDER_SIZE"
					:height="SQUARE_SIZE - SQUARE_BORDER_SIZE"
					:x="SQUARE_SIZE * 1.75"
					:y="SQUARE_SIZE * (index + 1)"
				/>
				<text
					:x="SQUARE_SIZE * 1.25"
					:y="SQUARE_SIZE * (curRangeColor.length + 2) - SQUARE_BORDER_SIZE"
				>
					{{ lo.more }}
				</text>
			</g>

			<g class="vch__year__wrapper" :transform="yearWrapperTransform" @mouseover="onHoverDay" @mouseleave="onMouseLeave">
				<g class="vch__month__wrapper"
				   v-for="(week, weekIndex) in heatmap.calendar"
				   :key="weekIndex"
				   :transform="getWeekPosition(weekIndex)"
				>
					<template v-for="(day, dayIndex) in week" :key="dayIndex">
						<rect class="vch__day__square"
							  v-if="day.date < now"
							  :rx="round"
							  :ry="round"
							  :transform="getDayPosition(dayIndex)"
							  :width="SQUARE_SIZE - SQUARE_BORDER_SIZE"
							  :height="SQUARE_SIZE - SQUARE_BORDER_SIZE"
							  :style="{ fill: curRangeColor[day.colorIndex] }"
							  :data-week-index="weekIndex"
							  :data-day-index="dayIndex"
							  @click="$emit('dayClick', day)"
						/>
					</template>
				</g>
			</g>
		</svg>
		<div class="vch__legend">
			<slot name="legend">
				<div class="vch__legend-left">
					<slot name="vch__legend-left"></slot>
				</div>
				<div class="vch__legend-right">
					<slot name="legend-right">
						<div class="vch__legend">
							<div>{{ lo.less }}</div>
							<svg v-if="!vertical" class="vch__external-legend-wrapper" :viewBox="legendViewbox" :height="SQUARE_SIZE - SQUARE_BORDER_SIZE">
								<g class="vch__legend__wrapper"  @mouseover="initTippyLazy">
									<rect class="vch__legend__square"
										v-for="(color, index) in curRangeColor"
										:key="index"
										:rx="round"
										:ry="round"
										:style="{ fill: color }"
										:width="SQUARE_SIZE - SQUARE_BORDER_SIZE"
										:height="SQUARE_SIZE - SQUARE_BORDER_SIZE"
										:x="SQUARE_SIZE * index"
                                        :colorIndex="index"
									/>
								</g>
							</svg>
							<div>{{ lo.more }}</div>
						</div>
					</slot>
				</div>
			</slot>
		</div>
	</div>
</template>

<script lang="ts">
	import { defineComponent, nextTick, onBeforeUnmount, onMounted, PropType, ref, toRef, toRefs, watch } from 'vue';
	import { CalendarItem, Heatmap, Locale, Month, TooltipFormatter, Value } from '@/components/Heatmap';
	import tippy, { createSingleton, CreateSingletonInstance, CreateSingletonProps, Instance } from 'tippy.js';
	import 'tippy.js/dist/tippy.css';
	import 'tippy.js/dist/svg-arrow.css';

	export default /*#__PURE__*/defineComponent({
		name : 'CalendarHeatmap',
		props: {
			endDate         : {
				required: true
			},
            startWeekday: {
                type: Number,
                default: 0, // 0 for Sunday, 1 for Monday, etc.
            },
			max             : {
				type: Number
			},
			rangeColor      : {
				type: Array as PropType<string[]>
			},
			values          : {
				type    : Array as PropType<Value[]>,
				required: true
			},
			locale          : {
				type: Object as PropType<Partial<Locale>>
			},
			tooltip         : {
				type   : Boolean,
				default: true
			},
			tooltipUnit     : {
				type   : String,
				default: Heatmap.DEFAULT_TOOLTIP_UNIT
			},
			tooltipFormatter: {
				type: Function as PropType<TooltipFormatter>
			},
			vertical        : {
				type   : Boolean,
				default: false
			},
			noDataText      : {
				type   : [ Boolean, String ],
				default: null
			},
			round           : {
				type   : Number,
				default: 0
			},
			darkMode        : Boolean,
            highlightedDay  : {
                type: Date as PropType<Date | null>,
                default: null,
            },
            showTooltipOnExternalHighlight: {
                type: Boolean,
                default: false,
            }
		},
		emits: [ 'dayClick', 'dayHover' ],
		setup(props) {

			const SQUARE_BORDER_SIZE          = Heatmap.SQUARE_SIZE / 5,
				  SQUARE_SIZE                 = Heatmap.SQUARE_SIZE + SQUARE_BORDER_SIZE,
				  LEFT_SECTION_WIDTH          = Math.ceil(Heatmap.SQUARE_SIZE * 2.5),
				  RIGHT_SECTION_WIDTH         = SQUARE_SIZE * 3,
				  TOP_SECTION_HEIGHT          = Heatmap.SQUARE_SIZE + (Heatmap.SQUARE_SIZE / 2),
				  BOTTOM_SECTION_HEIGHT       = Heatmap.SQUARE_SIZE + (Heatmap.SQUARE_SIZE / 2),
				  yearWrapperTransform        = `translate(${LEFT_SECTION_WIDTH}, ${TOP_SECTION_HEIGHT})`,
                  DAYS_IN_WEEK                = Heatmap.DAYS_IN_WEEK,

				  svg                         = ref<null | SVGElement>(null),
				  now                         = ref(new Date()),
				  heatmap                     = ref(new Heatmap(props.endDate as Date, props.values, props.max, props.rangeColor, props.startWeekday)),

				  width                       = ref(0),
				  height                      = ref(0),
				  viewbox                     = ref('0 0 0 0'),
				  legendViewbox               = ref('0 0 0 0'),
				  daysLabelWrapperTransform   = ref(''),
				  monthsLabelWrapperTransform = ref(''),
				  legendWrapperTransform      = ref(''),
				  lo                          = ref<Locale>({} as any),
				  rangeColor                  = ref<string[]>(props.rangeColor || (props.darkMode ? Heatmap.DEFAULT_RANGE_COLOR_DARK : Heatmap.DEFAULT_RANGE_COLOR_LIGHT))
                  ;

			const { values, tooltipUnit, tooltipFormatter, noDataText, max, vertical, locale, highlightedDay, showTooltipOnExternalHighlight, startWeekday } = toRefs(props),
				  tippyInstances                                                               = new Map<HTMLElement, Instance>();

			

			function tooltipOptions(day: CalendarItem) {
				if (props.tooltip) {
					if (day.count !== undefined) {
						if (props.tooltipFormatter) {
							return props.tooltipFormatter(day, props.tooltipUnit!);
						}
						return `<b>${day.count} ${props.tooltipUnit}</b> ${lo.value.on} ${lo.value.months[ day.date.getMonth() ]} ${day.date.getDate()}, ${day.date.getFullYear()}`;
					} else if (props.noDataText) {
						return `${props.noDataText}`;
					} else if (props.noDataText !== false) {
						return `<b>No ${props.tooltipUnit}</b> ${lo.value.on} ${lo.value.months[ day.date.getMonth() ]} ${day.date.getDate()}, ${day.date.getFullYear()}`;
					}
				}
				return undefined;
			}

			function getWeekPosition(index: number) {
				if (props.vertical) {
					return `translate(0, ${(SQUARE_SIZE * heatmap.value.weekCount) - ((index + 1) * SQUARE_SIZE)})`;
				}
				return `translate(${index * SQUARE_SIZE}, 0)`;
			}

			function getDayPosition(index: number) {
                if (props.vertical) {
                    return `translate(${index * SQUARE_SIZE}, 0)`;
                }
				return `translate(0, ${index * SQUARE_SIZE})`;
            }

			function getMonthLabelPosition(month: Month) {
                if (props.vertical) {
                    return { x: 3, y: (SQUARE_SIZE * heatmap.value.weekCount) - (SQUARE_SIZE * (month.index)) - (SQUARE_SIZE / 4) };
                }
				return { x: SQUARE_SIZE * month.index, y: SQUARE_SIZE - SQUARE_BORDER_SIZE };
            }

			watch([ toRef(props, 'rangeColor'), toRef(props, 'darkMode') ], ([ rc, dm ]) => {
				rangeColor.value = rc || (dm ? Heatmap.DEFAULT_RANGE_COLOR_DARK : Heatmap.DEFAULT_RANGE_COLOR_LIGHT);
			});

			watch(vertical, v => {
				if (v) {
					width.value                       = LEFT_SECTION_WIDTH + (SQUARE_SIZE * Heatmap.DAYS_IN_WEEK) + RIGHT_SECTION_WIDTH;
					height.value                      = TOP_SECTION_HEIGHT + (SQUARE_SIZE * heatmap.value.weekCount) + SQUARE_BORDER_SIZE;
					daysLabelWrapperTransform.value   = `translate(${LEFT_SECTION_WIDTH}, 0)`;
					monthsLabelWrapperTransform.value = `translate(0, ${TOP_SECTION_HEIGHT})`;
				} else {
					width.value                       = LEFT_SECTION_WIDTH + (SQUARE_SIZE * heatmap.value.weekCount) + SQUARE_BORDER_SIZE;
					height.value                      = TOP_SECTION_HEIGHT + (SQUARE_SIZE * Heatmap.DAYS_IN_WEEK);
					daysLabelWrapperTransform.value   = `translate(0, ${TOP_SECTION_HEIGHT})`;
					monthsLabelWrapperTransform.value = `translate(${LEFT_SECTION_WIDTH}, 0)`;
				}
			}, { immediate: true });

			watch([ width, height ], ([ w, h ]) => (viewbox.value = ` 0 0 ${w} ${h}`), { immediate: true });
			watch([ width, height, rangeColor ], ([ w, h, rc ]) => {
				legendWrapperTransform.value = vertical.value
					? `translate(${LEFT_SECTION_WIDTH + (SQUARE_SIZE * Heatmap.DAYS_IN_WEEK)}, ${TOP_SECTION_HEIGHT})`
					: `translate(${w - (SQUARE_SIZE * rc.length) - 30}, ${h - BOTTOM_SECTION_HEIGHT})`;
			}, { immediate: true });

			watch(locale, l => (lo.value = l ? { ...Heatmap.DEFAULT_LOCALE, ...l } : Heatmap.DEFAULT_LOCALE), { immediate: true });
			watch(rangeColor, rc => (legendViewbox.value = `0 0 ${(Heatmap.SQUARE_SIZE + 2) * (rc.length) - 2} ${Heatmap.SQUARE_SIZE}`), { immediate: true });

			watch(
				[ values, tooltipUnit, tooltipFormatter, noDataText, max, rangeColor, startWeekday ],
				() => {
					heatmap.value = new Heatmap(props.endDate as Date, props.values, props.max, rangeColor.value, props.startWeekday);
					tippyInstances.forEach((item) => item.destroy());
					nextTick(initTippy);
				}
			);

            // watch for highlighted day and show border
            let lastHighlightedDay: Date | null = null;
            watch([highlightedDay], (v) => {
                const day = v[0];

                // remove border from last highlighted day
                if (lastHighlightedDay) {
                    const weekIndex = heatmap.value.calendar.findIndex((week) => week.some((d) => d.date.getTime() === lastHighlightedDay!.getTime()));
                    if (weekIndex !== -1) {
                        const dayIndex = heatmap.value.calendar[ weekIndex ].findIndex((d) => d.date.getTime() === lastHighlightedDay!.getTime());
                        if (dayIndex !== -1) {
                            const square = svg.value?.querySelector(`.vch__day__square[data-week-index="${weekIndex}"][data-day-index="${dayIndex}"]`);
                            if (square) {
                                square.classList.remove('hover');
                            }
                        }
                    }
                    if(showTooltipOnExternalHighlight)
                    {
                        hideTippy();
                    }
                }
                lastHighlightedDay = day;

                if (day && day != null) {
                    const weekIndex = heatmap.value.calendar.findIndex((week) => week.some((d) => d.date.getTime() === day.getTime()));
                    if (weekIndex !== -1) {
                        const dayIndex = heatmap.value.calendar[ weekIndex ].findIndex((d) => d.date.getTime() === day.getTime());
                        if (dayIndex !== -1) {
                            const square = svg.value?.querySelector(`.vch__day__square[data-week-index="${weekIndex}"][data-day-index="${dayIndex}"]`);
                            if (square) {
                                square.classList.add('hover');
                                if(showTooltipOnExternalHighlight)
                                {
                                    const tooltip = tooltipOptions(heatmap.value.calendar[ weekIndex ][ dayIndex ]);
                                    if (tooltip) {
                                        nextTick(() => showTippy(square as HTMLElement, tooltip));
                                    }
                                }
                            }
                        }
                    }

                }
            });

			onMounted(initTippy);
			onBeforeUnmount(() => {
				tippySingleton?.destroy();
				tippyInstances.forEach((item) => item.destroy());
			});

            // get day from event and return it as Date
            function getDayFromEvent(e: MouseEvent): Date | null {
                if (e.target && (e.target as HTMLElement).classList.contains('vch__day__square')) {
                    const weekIndex = Number((e.target as HTMLElement).dataset.weekIndex),
                          dayIndex  = Number((e.target as HTMLElement).dataset.dayIndex);

                    if (!isNaN(weekIndex) && !isNaN(dayIndex)) {
                        return heatmap.value.calendar[ weekIndex ][ dayIndex ].date;
                    }
                }
                return null;
            }

            let tippySingleton: CreateSingletonInstance;

			function initTippy() {
				tippyInstances.clear();
				if (tippySingleton) {
					tippySingleton.setInstances(Array.from(tippyInstances.values()));
				} else {
					tippySingleton = createSingleton(Array.from(tippyInstances.values()), {
						overrides     : [],
						moveTransition: 'transform 0.1s ease-out',
						allowHTML     : true
					});
				}
			}

            // show tippy on html event with given content
            function showTippy(element : HTMLElement, content: string) {
                if (!tippySingleton) {
                    console.log("no singleton");
                    return;
                }

                let instance = tippyInstances.get(element);
                if (instance) {
                    instance.setContent(content);
                } else if (!instance) {
                    instance = tippy(element, { content, allowHTML : true } as any);
                    tippyInstances.set(element, instance);
                    tippySingleton.setInstances(Array.from(tippyInstances.values()));
                }

                tippySingleton.show(instance);
            }

            function hideTippy() {
                tippySingleton.hide();
            }

			function initTippyLazy(e: MouseEvent) {

				if (tippySingleton
					&& e.target
					&& (e.target as HTMLElement).classList.contains('vch__day__square')
					&& (e.target as HTMLElement).dataset.weekIndex !== undefined
					&& (e.target as HTMLElement).dataset.dayIndex !== undefined
				) {

					const weekIndex = Number((e.target as HTMLElement).dataset.weekIndex),
						  dayIndex  = Number((e.target as HTMLElement).dataset.dayIndex);

					if (!isNaN(weekIndex) && !isNaN(dayIndex)) {

						const tooltip = tooltipOptions(heatmap.value.calendar[ weekIndex ][ dayIndex ]);
						if (tooltip) {
                            showTippy(e.target as HTMLElement, tooltip);
						}
					}
				}
                // elseif branch for legend
                else if(tippySingleton
                    && e.target
                    && (e.target as HTMLElement).classList.contains('vch__legend__square')
                ) {
                    
                    let attr = (e.target as HTMLElement).getAttribute('colorIndex');
                    if(attr === null) {
                        return;
                    }
                    const index = parseInt(attr);
                    const rangeValueLower = Math.round((heatmap.value.max / rangeColor.value.length) * index);
                    const rangeValueUpper = Math.round((heatmap.value.max / rangeColor.value.length ) * (index+ 1));

                    // TODO use tooltipFormatter if available
                    let content = '';
                    //if (props.tooltipFormatter) {
                    //    content = props.tooltipFormatter({ count: rangeValue, date: new Date(), colorIndex: index }, props.tooltipUnit!);
                    //} else {
                        content = `<b>${rangeValueLower} to ${rangeValueUpper} ${props.tooltipUnit}</b>`;
                    //}

                    showTippy(e.target as HTMLElement, content);
                }
			}

			return {
				SQUARE_BORDER_SIZE, SQUARE_SIZE, LEFT_SECTION_WIDTH, RIGHT_SECTION_WIDTH, TOP_SECTION_HEIGHT, BOTTOM_SECTION_HEIGHT, DAYS_IN_WEEK,
				svg, heatmap, now, width, height, viewbox, daysLabelWrapperTransform, monthsLabelWrapperTransform, yearWrapperTransform, legendWrapperTransform,
				lo, legendViewbox, curRangeColor: rangeColor,
				getWeekPosition, getDayPosition, getMonthLabelPosition, initTippyLazy, getDayFromEvent
			};
		},
        methods: {
            onHoverDay(e: MouseEvent) {
                this.initTippyLazy(e);

                const day = this.getDayFromEvent(e);
                if (day) {
                    this.$emit('dayHover', day);
                }
            },
            onMouseLeave() {
                this.$emit('dayHover', null);
            }
        }
	});
</script>

<style lang="scss">

	.vch__container {
		.vch__legend {
			display: flex;
			justify-content: space-between;
			align-items: center;
		}

		.vch__external-legend-wrapper {
			margin: 0 8px;
		}
	}

	svg.vch__wrapper {
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
		line-height: 10px;
		width: 100%;

		.vch__months__labels__wrapper text.vch__month__label {
			font-size: 10px;
		}

		.vch__days__labels__wrapper text.vch__day__label,
		.vch__legend__wrapper text {
			font-size: 9px;
		}

		text.vch__month__label,
		text.vch__day__label,
		.vch__legend__wrapper text {
			fill: #767676;
		}

		rect.vch__day__square:hover,
        rect.vch__day__square.hover {
			stroke: #555;
			stroke-width: 2px;
			paint-order: stroke;
		}

		rect.vch__day__square:focus {
			outline: none;
		}

		&.dark-mode {
			text.vch__month__label,
			text.vch__day__label,
			.vch__legend__wrapper text {
				fill: #fff;
			}
		}
	}

</style>
