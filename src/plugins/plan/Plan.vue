<template>
<div ref="plan"
     class="c-plan"
>
    <template v-if="viewBounds && !options.compact">
        <swim-lane>
            <template slot="label">{{ timeSystem.name }}</template>
            <timeline-axis
                slot="object"
                :bounds="viewBounds"
                :time-system="timeSystem"
                :content-height="height"
                :rendering-engine="renderingEngine"
            />
        </swim-lane>
    </template>
    <div ref="planHolder"
         class="c-plan__contents u-contents"
    >
    </div>
</div>
</template>

<script>
import * as d3Selection from 'd3-selection';
import * as d3Scale from 'd3-scale';
import TimelineAxis from "../../ui/components/TimeSystemAxis.vue";
import SwimLane from "@/ui/components/swim-lane/SwimLane.vue";
import Vue from "vue";

//TODO: UI direction needed for the following property values
const PADDING = 1;
const OUTER_TEXT_PADDING = 12;
const INNER_TEXT_PADDING = 17;
const TEXT_LEFT_PADDING = 5;
const ROW_PADDING = 12;
const RESIZE_POLL_INTERVAL = 200;
const ROW_HEIGHT = 25;
const LINE_HEIGHT = 12;
const MAX_TEXT_WIDTH = 300;
const EDGE_ROUNDING = 10;
const DEFAULT_COLOR = 'yellow';
const DEFAULT_TEXT_COLOR = 'white';

export default {
    components: {
        TimelineAxis,
        SwimLane
    },
    inject: ['openmct', 'domainObject'],
    props: {
        options: {
            type: Object,
            default() {
                return {
                    compact: false
                };
            }
        },
        renderingEngine: {
            type: String,
            default() {
                return 'svg';
            }
        }
    },
    data() {
        return {
            viewBounds: undefined,
            timeSystem: undefined,
            height: 0
        };
    },
    mounted() {
        this.validateJSON(this.domainObject.selectFile.body);

        this.canvas = this.$refs.plan.appendChild(document.createElement('canvas'));
        this.canvas.height = 0;
        this.canvasContext = this.canvas.getContext('2d');

        this.setDimensions();
        this.updateViewBounds();
        this.openmct.time.on("timeSystem", this.setScaleAndPlotActivities);
        this.openmct.time.on("bounds", this.updateViewBounds);
        this.resizeTimer = setInterval(this.resize, RESIZE_POLL_INTERVAL);
        this.unlisten = this.openmct.objects.observe(this.domainObject, '*', this.observeForChanges);
    },
    beforeDestroy() {
        clearInterval(this.resizeTimer);
        this.openmct.time.off("timeSystem", this.setScaleAndPlotActivities);
        this.openmct.time.off("bounds", this.updateViewBounds);
        if (this.unlisten) {
            this.unlisten();
        }
    },
    methods: {
        observeForChanges(mutatedObject) {
            this.validateJSON(mutatedObject.selectFile.body);
            this.setScaleAndPlotActivities();
        },
        resize() {
            let clientWidth = this.getClientWidth();
            if (clientWidth !== this.width) {
                this.setDimensions();
                this.updateViewBounds();
            }
        },
        getClientWidth() {
            let clientWidth = this.$refs.plan.clientWidth;

            if (!clientWidth) {
                //this is a hack - need a better way to find the parent of this component
                let parent = this.openmct.layout.$refs.browseObject.$el;
                if (parent) {
                    clientWidth = parent.getBoundingClientRect().width;
                }
            }

            return clientWidth - 200;
        },
        validateJSON(jsonString) {
            try {
                this.json = JSON.parse(jsonString);
            } catch (e) {
                return false;
            }

            return true;
        },
        updateViewBounds() {
            this.viewBounds = this.openmct.time.bounds();
            if (this.timeSystem === undefined) {
                this.timeSystem = this.openmct.time.timeSystem();
            }

            this.setScaleAndPlotActivities();
        },
        setScaleAndPlotActivities(timeSystem) {
            if (timeSystem !== undefined) {
                this.timeSystem = timeSystem;
            }

            this.setScale(this.timeSystem);
            this.clearPreviousActivities();
            if (this.xScale) {
                this.calculatePlanLayout();
                this.drawPlan();
            }
        },
        clearPreviousActivities() {
            d3Selection.selectAll(".c-plan__contents > div").remove();
        },
        setDimensions() {
            const planHolder = this.$refs.plan;
            this.width = this.getClientWidth();

            this.height = Math.round(planHolder.getBoundingClientRect().height);
        },
        setScale(timeSystem) {
            if (!this.width) {
                return;
            }

            if (timeSystem === undefined) {
                timeSystem = this.openmct.time.timeSystem();
            }

            if (timeSystem.isUTCBased) {
                this.xScale = d3Scale.scaleUtc();
                this.xScale.domain(
                    [new Date(this.viewBounds.start), new Date(this.viewBounds.end)]
                );
            } else {
                this.xScale = d3Scale.scaleLinear();
                this.xScale.domain(
                    [this.viewBounds.start, this.viewBounds.end]
                );
            }

            this.xScale.range([PADDING, this.width - PADDING * 2]);
        },
        isActivityInBounds(activity) {
            return (activity.start < this.viewBounds.end) && (activity.end > this.viewBounds.start);
        },
        getTextWidth(name) {
            let metrics = this.canvasContext.measureText(name);

            return parseInt(metrics.width, 10);
        },
        sortFn(a, b) {
            const numA = parseInt(a, 10);
            const numB = parseInt(b, 10);
            if (numA > numB) {
                return 1;
            }

            if (numA < numB) {
                return -1;
            }

            return 0;
        },
        // Get the row where the next activity will land.
        getRowForActivity(rectX, width, activitiesByRow) {
            let currentRow;
            let sortedActivityRows = Object.keys(activitiesByRow).sort(this.sortFn);

            function getOverlap(rects) {
                return rects.every(rect => {
                    const { start, end } = rect;
                    const calculatedEnd = rectX + width;
                    const hasOverlap = (rectX >= start && rectX <= end) || (calculatedEnd >= start && calculatedEnd <= end) || (rectX <= start && calculatedEnd >= end);

                    return !hasOverlap;
                });
            }

            for (let i = 0; i < sortedActivityRows.length; i++) {
                let row = sortedActivityRows[i];
                if (getOverlap(activitiesByRow[row])) {
                    currentRow = row;
                    break;
                }
            }

            if (currentRow === undefined && sortedActivityRows.length) {
                let row = parseInt(sortedActivityRows[sortedActivityRows.length - 1], 10);
                currentRow = row + ROW_HEIGHT + ROW_PADDING;
            }

            return (currentRow || 0);
        },
        calculatePlanLayout() {
            let groups = Object.keys(this.json);
            this.groupActivities = {};

            groups.forEach((key, index) => {
                let activitiesByRow = {};
                let currentRow = 0;

                let activities = this.json[key];
                activities.forEach((activity) => {
                    if (this.isActivityInBounds(activity)) {
                        const currentStart = Math.max(this.viewBounds.start, activity.start);
                        const currentEnd = Math.min(this.viewBounds.end, activity.end);
                        const rectX = this.xScale(currentStart);
                        const rectY = this.xScale(currentEnd);
                        const rectWidth = rectY - rectX;

                        const activityNameWidth = this.getTextWidth(activity.name) + TEXT_LEFT_PADDING;
                        //TODO: Fix bug for SVG where the rectWidth is not proportional to the canvas measuredWidth of the text
                        const activityNameFitsRect = (rectWidth >= activityNameWidth);
                        const textStart = (activityNameFitsRect ? rectX : rectY) + TEXT_LEFT_PADDING;

                        let textLines = this.getActivityDisplayText(this.canvasContext, activity.name, activityNameFitsRect);
                        const textWidth = textStart + this.getTextWidth(textLines[0]) + TEXT_LEFT_PADDING;

                        if (activityNameFitsRect) {
                            currentRow = this.getRowForActivity(rectX, rectWidth, activitiesByRow);
                        } else {
                            currentRow = this.getRowForActivity(rectX, textWidth, activitiesByRow);
                        }

                        let textY = parseInt(currentRow, 10) + (activityNameFitsRect ? INNER_TEXT_PADDING : OUTER_TEXT_PADDING);

                        if (!activitiesByRow[currentRow]) {
                            activitiesByRow[currentRow] = [];
                        }

                        activitiesByRow[currentRow].push({
                            activity: {
                                color: activity.color || DEFAULT_COLOR,
                                textColor: activity.textColor || DEFAULT_TEXT_COLOR,
                                name: activity.name,
                                exceeds: {
                                    start: this.xScale(this.viewBounds.start) > this.xScale(activity.start),
                                    end: this.xScale(this.viewBounds.end) < this.xScale(activity.end)
                                }
                            },
                            textLines: textLines,
                            textStart: textStart,
                            textY: textY,
                            start: rectX,
                            end: activityNameFitsRect ? rectY : textStart + textWidth,
                            rectWidth: rectWidth
                        });
                    }
                });
                this.groupActivities[key] = {
                    heading: key,
                    activitiesByRow
                };
            });
        },
        getActivityDisplayText(context, text, activityNameFitsRect) {
        //TODO: If the activity start is less than viewBounds.start then the text should be cropped on the left/should be off-screen)
            let words = text.split(' ');
            let line = '';
            let activityText = [];
            let rows = 1;

            for (let n = 0; (n < words.length) && (rows <= 2); n++) {
                let testLine = line + words[n] + ' ';
                let metrics = context.measureText(testLine);
                let testWidth = metrics.width;
                if (!activityNameFitsRect && (testWidth > MAX_TEXT_WIDTH && n > 0)) {
                    activityText.push(line);
                    line = words[n] + ' ';
                    testLine = line + words[n] + ' ';
                    rows = rows + 1;
                }

                line = testLine;
            }

            return activityText.length ? activityText : [line];
        },
        getGroupContainer(activityRows, heading) {
            let svgHeight = 30;
            let svgWidth = 200;

            const rows = Object.keys(activityRows);
            const isNested = this.options.isChildObject;

            if (rows.length) {
                const lastActivityRow = rows[rows.length - 1];
                svgHeight = parseInt(lastActivityRow, 10) + ROW_HEIGHT;
                svgWidth = this.width;
            }

            let component = new Vue({
                components: {
                    SwimLane
                },
                data() {
                    return {
                        heading,
                        isNested,
                        height: svgHeight,
                        width: svgWidth
                    };
                },
                template: `<swim-lane :is-nested="isNested"><template slot="label">{{heading}}</template><template slot="object"><svg :height="height" :width="width"></svg></template></swim-lane>`
            });

            this.$refs.planHolder.appendChild(component.$mount().$el);

            let groupLabel = component.$el.querySelector('div:nth-child(1)');
            let groupSVG = component.$el.querySelector('svg');

            return {
                groupLabel,
                groupSVG
            };
        },
        drawPlan() {

            Object.keys(this.groupActivities).forEach((group, index) => {
                const activitiesByRow = this.groupActivities[group].activitiesByRow;
                const heading = this.groupActivities[group].heading;
                const groupElements = this.getGroupContainer(activitiesByRow, heading);
                let groupSVG = groupElements.groupSVG;

                let activityRows = Object.keys(activitiesByRow);
                if (activityRows.length <= 0) {
                    this.plotNoItems(groupSVG);
                }

                activityRows.forEach((row) => {
                    const items = activitiesByRow[row];
                    items.forEach(item => {
                    //TODO: Don't draw the left-border of the rectangle if the activity started before viewBounds.start
                        this.plotActivity(item, parseInt(row, 10), groupSVG);
                    });
                });

            });
        },
        plotNoItems(svgElement) {
            let textElement = document.createElementNS('http://www.w3.org/2000/svg', 'text');
            this.setNSAttributesForElement(textElement, {
                x: "10",
                y: "20",
                class: "no-activities"
            });
            textElement.innerHTML = 'No activities at this time';

            svgElement.appendChild(textElement);
        },
        setNSAttributesForElement(element, attributes) {
            Object.keys(attributes).forEach((key) => {
                element.setAttributeNS(null, key, attributes[key]);
            });
        },
        // Experimental for now - unused
        addForeignElement(svgElement, label, x, y) {
            let foreign = document.createElementNS('http://www.w3.org/2000/svg', "foreignObject");
            this.setNSAttributesForElement(foreign, {
                width: String(MAX_TEXT_WIDTH),
                height: String(LINE_HEIGHT * 2),
                x: x,
                y: y
            });

            let textEl = document.createElement('div');
            let textNode = document.createTextNode(label);
            textEl.appendChild(textNode);

            foreign.appendChild(textEl);

            svgElement.appendChild(foreign);
        },
        plotActivity(item, row, svgElement) {
            const activity = item.activity;
            let width = item.rectWidth;
            let rectElement = document.createElementNS('http://www.w3.org/2000/svg', 'rect');

            if (item.activity.exceeds.start) {
                width = width + EDGE_ROUNDING;
            }

            if (item.activity.exceeds.end) {
                width = width + EDGE_ROUNDING;
            }

            this.setNSAttributesForElement(rectElement, {
                class: 'activity-bounds',
                x: item.activity.exceeds.start ? item.start - EDGE_ROUNDING : item.start,
                y: row,
                rx: EDGE_ROUNDING,
                width: width,
                height: String(ROW_HEIGHT),
                fill: activity.color
            });

            svgElement.appendChild(rectElement);

            item.textLines.forEach((line, index) => {
                let textElement = document.createElementNS('http://www.w3.org/2000/svg', 'text');
                this.setNSAttributesForElement(textElement, {
                    class: 'activity-label',
                    x: item.textStart,
                    y: item.textY + (index * LINE_HEIGHT),
                    fill: activity.textColor
                });

                const textNode = document.createTextNode(line);
                textElement.appendChild(textNode);
                svgElement.appendChild(textElement);
            });
            // this.addForeignElement(svgElement, activity.name, item.textStart, item.textY - LINE_HEIGHT);
        }
    }
};
</script>
