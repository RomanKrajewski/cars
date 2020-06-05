<template>
    <div>
        <h1>{{ msg }}</h1>
        <div>
            <div id="chartComponent"></div>
            <v-select style="width:20%; margin-left:auto; margin-right:0" :clearable=false v-model="selectedXAxes"
                      v-on:input="updateChart"
                      :options="['Verbrauch','Zylinder','Hubraum','PS','Gewicht','Beschleunigung','Baujahr']"></v-select>
        </div>
    </div>
</template>

<script>
    import * as d3 from "d3"
    import Vue from "vue"
    import 'vue-select/dist/vue-select.css';
    import vSelect from "vue-select"

    Vue.component("v-select", vSelect)

    export default {
        name: 'Chart',
        props: {
            data: Array
        },
        computed: {
            chartData: function () {
                return this.data
                    .map((car, i) => {
                        return {
                            index: i,
                            manufacturer: car.Hersteller,
                            region: car.Herkunft,
                            x: Number(car[this.selectedXAxes]),
                            y: Number(car.Verbrauch)
                        }
                    })
                    .filter(entry => !isNaN(entry.x) && !isNaN(entry.y))
            },
            x: function () {
                return d3.scaleLinear()
                    .domain(d3.extent(this.chartData, d => d.x)).nice()
                    .range([this.margin.left, this.width - this.margin.right])
            },
            y: function () {
                return d3.scaleLinear()
                    .domain(d3.extent(this.chartData, d => d.y)).nice()
                    .range([this.height - this.margin.bottom, this.margin.top])
            }
        },
        data() {
            return {
                msg: "🚗 🚙 🚓 🚕 🚙 🚓 🚗 🚕",
                height: 600,
                width: 800,
                selectedXAxes: "Baujahr",
                margin: {top: 25, right: 20, bottom: 35, left: 40},
                color: Object,
                shape: Object,
                componentDiv: Object,
                tooltip: Object,
                tooltipRect: Object,
                drawnXAxis: Object,
                drawnYAxis: Object,
                drawnGlyphs: Object,
            }
        },
        methods: {
            xAxis(g) {
                return g.attr("transform", `translate(0,${this.height - this.margin.bottom})`)
                    .transition().duration(1000).ease(d3.easeLinear)
                    .call(d3.axisBottom(this.x).ticks(this.width / 80))
            },
            yAxis(g) {
                return g.attr("transform", `translate(${this.margin.left},0)`)
                    .transition().duration(2000)
                    .call(d3.axisLeft(this.y))
            },
            generateChart() {
                this.color = d3.scaleOrdinal().range([...Array(30).keys()].map(i => `hsl(${i * 12}, 100%, 60%)`));
                this.shape = d3.scaleOrdinal(this.chartData.map(d => d.category), d3.symbols.map(s => d3.symbol().type(s)()));
                this.componentDiv = d3.select("#chartComponent")
                const svg = this.componentDiv
                    .append("svg")
                    .attr("viewBox", [0, 0, this.width, this.height]);

                this.tooltipRect = svg.append("rect")
                    .attr("fill", "white")
                    .attr("stroke", "black")
                this.tooltip = svg.append("text")
                this.drawnXAxis = svg.append("g")
                this.drawnYAxis = svg.append("g")
                this.drawnGlyphs = svg.append("g")
                    .attr("stroke-width", 0.5)
                    .attr("font-family", "sans-serif")
                    .attr("font-size", 50)

                this.tooltipRect.raise();
                this.tooltip.raise();
            },
            updateChart() {
                this.drawnYAxis.call(this.xAxis);
                this.drawnXAxis.call(this.yAxis);
                this.drawnGlyphs.selectAll("path")
                    .data(this.chartData, d => d.index)
                    .join("path")
                    .on("mouseover", d => {
                        this.tooltip.attr("visibility", "visible")
                        this.tooltipRect.attr("visibility", "visible")
                        this.tooltip.text(this.data[d.index].Model);
                        const bb = this.tooltip.node().getBBox();
                        this.tooltip.attr("x", (this.x(d.x) - bb.width / 2)).attr("y", (this.y(d.y) - bb.height / 2));
                        this.tooltipRect.attr("x", (this.x(d.x) - bb.width * 0.5)).attr("y", (this.y(d.y) - bb.height * 1.25)).attr("width", bb.width).attr("height", bb.height);
                        d3.select(d3.event.target).attr("stroke", "black").raise()
                    })
                    .on("mouseout", () => {
                        this.tooltip.attr("visibility", "hidden")
                        this.tooltipRect.attr("visibility", "hidden")
                        d3.select(d3.event.target).attr("stroke", "none")
                    })
                    .transition().duration(1000).ease(d3.easeCubic)
                    .attr("transform", d => `translate(${this.x(d.x)},${this.y(d.y)})`)
                    .attr("fill", d => this.color(d.manufacturer))
                    .attr("d", d => this.shape(d.region));
            }
        },
        mounted() {
            this.generateChart();
            this.updateChart();
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    h3 {
        margin: 40px 0 0;
    }

    ul {
        list-style-type: none;
        padding: 0;
    }

    li {
        display: inline-block;
        margin: 0 10px;
    }

    a {
        color: #42b983;
    }
</style>
