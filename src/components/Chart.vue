<template>
    <div style="display:flex; align-items:stretch">
        <div style="width:15%; display:flex; flex-direction:column; align-items:flex-start">
            <v-select style="width:100%"
                      :clearable=false
                      :searchable=false
                      v-model="selectedYAxes"
                      v-on:input="updateChart"
                      :options="['Verbrauch','Zylinder','Hubraum','PS','Gewicht','Beschleunigung','Baujahr']"></v-select>
            <v-select style="display: flex"
                      :clearable=false
                      :multiple=true
                      v-model="selectedManufacturers"
                      v-on:input="updateManufacturers"
                      :options="notSelectedManufacturers">
            </v-select>
        </div>
        <div id="chartComponent"></div>
        <div style="display:flex;flex-direction: column-reverse; width:15%; padding-bottom: 2em">
        <v-select style="width:100%;"
                  :clearable=false
                  :searchable=false
                  v-model="selectedXAxes"
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
                            y: Number(car[this.selectedYAxes])
                        }
                    })
                    .filter(entry => !isNaN(entry.x) && !isNaN(entry.y) && this.selectedManufacturers.includes(entry.manufacturer))
            },
            manufacturerList: function () {
                return [...new Set(this.data.map(car => car.Hersteller))]
            },
            notSelectedManufacturers: function () {
                return this.manufacturerList.filter(e => !this.selectedManufacturers.includes(e))
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
                height: 600,
                width: 800,
                manufacturerColorLimit: 10,
                selectedXAxes: "PS",
                selectedYAxes: "Verbrauch",
                selectedManufacturers: [],
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
                    .select(".domain").attr("stroke", "none")
            },
            yAxis(g) {
                return g.attr("transform", `translate(${this.margin.left},0)`)
                    .transition().duration(2000)
                    .call(d3.axisLeft(this.y))
                    .select(".domain").attr("stroke", "none")
            },
            generateChart() {
                this.manufacturerColor = d3.scaleOrdinal(d3.schemeTableau10);
                this.regionColor = d3.scaleOrdinal(d3.schemeTableau10);
                this.shape = d3.scaleOrdinal(this.chartData.map(d => d.region), ["symbolSquare", "symbolTriangle", "symbolCircle"].map(s => d3.symbol().type(d3[s])));
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
            getShape(d) {
                if (this.selectedManufacturers.length > this.manufacturerColorLimit) {
                    return d3.symbol().type(d3["symbolCircle"]).size("10")
                } else {
                    return this.shape(d.region).size("30")
                }
            },
            getColor(d) {
                if (this.selectedManufacturers.length > this.manufacturerColorLimit) {
                    return this.regionColor(d.region)
                } else {
                    return this.manufacturerColor(d.manufacturer)
                }
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
                        this.tooltip.text(`${this.data[d.index].Hersteller} ${this.data[d.index].Model}`);
                        const bb = this.tooltip.node().getBBox();
                        this.tooltip.attr("x", (this.x(d.x) - bb.width / 2)).attr("y", (this.y(d.y) - bb.height / 2));
                        this.tooltipRect.attr("x", (this.x(d.x) - bb.width * 0.5)).attr("y", (this.y(d.y) - bb.height * 1.25)).attr("width", bb.width).attr("height", bb.height);
                        d3.select(d3.event.target)
                            .attr("stroke", "black").raise()
                            .attr("d", d => this.getShape(d).size("100")())
                    })
                    .on("mouseout", () => {
                        this.tooltip.attr("visibility", "hidden")
                        this.tooltipRect.attr("visibility", "hidden")
                        d3.select(d3.event.target)
                            .attr("stroke", "none")
                            .attr("d", d => this.getShape(d)())
                    })
                    .attr("d", d => this.getShape(d)())
                    .transition().duration(1000).ease(d3.easeCubic)
                    .attr("transform", d => `translate(${this.x(d.x)},${this.y(d.y)})`)
                    .attr("fill", d => this.getColor(d));
            },
            updateManufacturers() {
                this.manufacturerColor.domain(this.manufacturerColor.domain().filter(e => this.selectedManufacturers.includes(e)))
                this.updateChart()
            }
        },
        mounted() {
            this.selectedManufacturers = this.manufacturerList
            this.generateChart();
            this.updateChart();
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    #chartComponent {
        width: 70%;
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
