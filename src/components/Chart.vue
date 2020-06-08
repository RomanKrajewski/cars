<template>
    <div style="display: flex; flex-direction: column">
        <div style="display:flex; align-items:stretch">
            <div style="width:15%; display:flex; flex-direction:column; align-items:flex-start">
                <v-select style="width:100%"
                          :clearable=false
                          :searchable=false
                          v-model="selectedYAxes"
                          v-on:input="updateChart"
                          :options="['Verbrauch','Zylinder','Hubraum','PS','Gewicht','Beschleunigung','Baujahr']"></v-select>
                <div style="width:100%; display:flex; flex-direction: column; padding-top:10px">
                    <input class="visHidden" v-for="region in regionList" :key="region" type="checkbox"
                           :id="`region_checkbox_${region}`" :value="region" v-model="selectedRegions">
                    <label style="width:100%; display: flex;justify-content: center; margin-left:0;margin-right: 0; padding-left: 0; padding-right: 0"
                           class="regionLabel noselect" v-for="region in regionList" :key="`${region}label`"
                           :for="`region_checkbox_${region}`">{{region}}
                        <svg viewBox="0 0 10 10" style="margin-left: 0.5em;align-self: center;" height="1em">
                            <path></path>
                        </svg>
                    </label>
                </div>
            </div>
            <div style="width:70%" id="chartComponent"/>
            <div style="display:flex;flex-direction: column; width:15%; padding-bottom: 2em">
                <h2>Visualisierung CarsDE</h2>
                <p class="like-label" v-if="!currentCar.Hersteller">Details on hover</p>
                <div class="like-label" style="display: flex; flex-direction: column" v-else>
                    <h2 style="margin-top:0; margin-bottom: 0">{{ `${capitalize(currentCar.Hersteller)}` }}</h2>
                    <h3 style="margin-top:0; margin-bottom: 0.5em;"> {{`${capitalize(currentCar.Model)}` }} </h3>
                    <div style="align-self: flex-start" v-for="key in detailKeys" :key="`car_detail_${key}`">
                        <p style="margin: 0"><strong>{{ key }}: </strong> {{ currentCar[key] }}</p>
                    </div>
                </div>
                <v-select style="width:100%; margin-bottom:0; margin-top:auto"
                          :clearable=false
                          :searchable=false
                          v-model="selectedXAxes"
                          v-on:input="updateChart"
                          :options="['Verbrauch','Zylinder','Hubraum','PS','Gewicht','Beschleunigung','Baujahr']"></v-select>
            </div>
        </div>
        <div style="display:flex; flex-wrap:wrap; width:70%; align-self:center; padding-left:40px">
            <input class="visHidden" v-for="manufacturer in manufacturerList" :key="manufacturer" type="checkbox"
                   :id="`man_checkbox_${manufacturer}`" :value="manufacturer" v-model="selectedManufacturers">
            <label class="manufacturerLabel noselect" v-for="manufacturer in manufacturerList"
                   :key="`${manufacturer}label`"
                   :for="`man_checkbox_${manufacturer}`">{{ capitalize(manufacturer) }}</label>
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
            detailKeys: function () {
                return Object.keys(this.currentCar).filter(e => !['Hersteller', 'Model'].includes(e))
            },
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
                    .filter(entry => !isNaN(entry.x) && !isNaN(entry.y) && this.selectedManufacturers.includes(entry.manufacturer) && this.selectedRegions.includes(entry.region))
            },
            manufacturerList: function () {
                return [...new Set(this.data.map(car => car.Hersteller))].sort()
            },
            regionList: function () {
                return [...new Set(this.data.map(car => car.Herkunft))].sort()
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
                height: 500,
                width: 800,
                manufacturerColorLimit: 10,
                selectedXAxes: "PS",
                selectedYAxes: "Verbrauch",
                selectedManufacturers: [],
                selectedRegions: [],
                margin: {top: 25, right: 20, bottom: 35, left: 40},
                regionColor: Object,
                currentCar: {},
                shape: Object,
                componentDiv: Object,
                manufacturerColors: Object,
                unusedManufacturerColors: Object,
                drawnXAxis: Object,
                drawnYAxis: Object,
                drawnGlyphs: Object,
            }
        },
        watch: {
            selectedManufacturers: function () {
                this.updateManufacturers()
            },
            selectedRegions: function () {
                this.updateRegions()
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
                    .transition().duration(1000).ease(d3.easeLinear)
                    .call(d3.axisLeft(this.y))
                    .select(".domain").attr("stroke", "none")
            },
            generateChart() {
                this.regionColor = d3.scaleOrdinal(d3.schemeTableau10);
                this.shape = d3.scaleOrdinal(this.chartData.map(d => d.region), ["symbolSquare", "symbolTriangle", "symbolCircle"].map(s => d3.symbol().type(d3[s])));
                this.componentDiv = d3.select("#chartComponent")
                const svg = this.componentDiv
                    .append("svg")
                    .attr("viewBox", [0, 0, this.width, this.height]);

                this.drawnXAxis = svg.append("g")
                this.drawnYAxis = svg.append("g")
                this.drawnGlyphs = svg.append("g")
                    .attr("stroke-width", 0.5)
                    .attr("font-family", "sans-serif")
                    .attr("font-size", 50)
            },
            getShape(d) {
                if (this.selectedManufacturers.length > this.manufacturerColorLimit) {
                    return d3.symbol().type(d3["symbolCircle"]).size("20")
                } else {
                    return this.shape(d.region).size("30")
                }
            },
            getColor(d) {
                if (!this.selectedManufacturers.includes(d.manufacturer)) {
                    return "#ffffff"
                }
                if (this.selectedManufacturers.length > this.manufacturerColorLimit) {
                    return this.regionColor(d.region)
                } else {
                    if (this.manufacturerColors.has(d.manufacturer)) {
                        return this.manufacturerColors.get(d.manufacturer)
                    } else {
                        let newColor = this.unusedManufacturerColors.pop()
                        this.manufacturerColors.set(d.manufacturer, newColor)
                        return newColor;
                    }
                }
            },
            updateChart() {

                this.drawnYAxis.call(this.xAxis);
                this.drawnXAxis.call(this.yAxis);

                this.drawnGlyphs.selectAll("path")
                    .data(this.chartData, d => d.index)
                    .join(enter => enter.append("path")
                            .on("mouseover", d => {
                                this.currentCar = this.data[d.index]
                                d3.select(d3.event.target)
                                    .attr("stroke", "black").raise()
                                    .transition().duration(100).ease(d3.easeBackOut.overshoot(3))
                                    .attr("d", d => this.getShape(d).size("100")())
                            })
                            .on("mouseout", () => {
                                this.currentCar = {}
                                d3.select(d3.event.target)
                                    .transition().delay(50).duration(50).ease(d3.easeBackIn.overshoot(3))
                                    .attr("stroke", "none")
                                    .attr("d", d => this.getShape(d)())
                            })
                            .attr("d", d => this.getShape(d)())
                            .attr("fill", d => this.getColor(d))
                            .call(enter => enter
                                .transition().duration(1000).ease(d3.easeCubic)
                                .attrTween("transform", d => {
                                    return d3.interpolateString(`translate(0,${this.height})`, `translate(${this.x(d.x)},${this.y(d.y)})`);
                                })),
                        update => update
                            .attr("d", d => this.getShape(d)())
                            .transition().duration(1000).ease(d3.easeCubic)
                            .attr("fill", d => this.getColor(d))
                            .attr("transform", d => `translate(${this.x(d.x)},${this.y(d.y)})`),
                        exit => exit
                            .call(exit => exit.transition().duration(500).ease(d3.easeCubic).attr("transform", `translate(${this.width}, 0)`)
                                .remove())
                    );
            },
            updateManufacturerLegend() {
                d3.selectAll(".manufacturerLabel")
                    .datum((d, i, n) => {
                        const car = this.data.find(e => e.Hersteller === d3.select(n[i]).text().trim().toLowerCase())
                        return this.getColor({manufacturer: car.Hersteller, region: car.Herkunft})
                    })
                    .each(function (d) {
                        const l = d3.hsl(d).l
                        const textColor = l > 0.65 ? "#2c3e50" : "white"
                        d3.select(this)
                            .transition().duration(500)
                            .style("background-color", d)
                            .style("color", textColor)
                    })
            },
            updateRegionLegend() {
                d3.selectAll(".regionLabel")
                    .datum((d, i, n) => {
                        const region = d3.select(n[i]).text().trim()
                        if (!this.selectedRegions.includes(region)) {
                            return {shape: this.getShape({region: region}).size(0), color: "#ffffff"}
                        }
                        if (this.selectedManufacturers.length > this.manufacturerColorLimit) {
                            return {shape: this.getShape({region: region}).size(40), color: this.regionColor(region)}
                        }
                        return {shape: this.getShape({region: region}).size(40), color: "#2c3e50"}
                    })
                    .each(function (d) {
                        const l = d3.hsl(d.color).l
                        const textColor = l > 0.65 ? "#2c3e50" : "white"
                        // const textColor = "white"
                        d3.select(this)
                            .selectAll("path")
                            .datum(function () {
                                return d
                            })
                            .attr("d", d => d.shape())
                            .attr("fill", textColor)
                            .attr("transform", "translate(5,5)")
                        d3.select(this).transition().duration(500)
                            .style("background-color", d.color)
                            .style("color", textColor)
                    })
            },
            updateRegions() {
                this.updateRegionLegend()
                this.updateChart()
            },
            updateManufacturers() {
                let unselectedManufacturer = Array.from(this.manufacturerColors.keys()).filter(e => !this.selectedManufacturers.includes(e))[0]
                if (this.manufacturerColors.has(unselectedManufacturer)) {
                    this.unusedManufacturerColors.push(this.manufacturerColors.get(unselectedManufacturer))
                    this.manufacturerColors.delete(unselectedManufacturer)
                }
                this.updateManufacturerLegend()
                this.updateRegionLegend()
                this.updateChart()
            },
            capitalize(string) {
                const safeString = string.toString()
                return safeString.charAt(0).toUpperCase() + safeString.slice(1);
            }
        }
        ,
        created() {
            this.manufacturerColors = new Map()
            this.unusedManufacturerColors = d3.schemeCategory10

            this.selectedManufacturers = this.manufacturerList.slice(0, 11)
            this.selectedRegions = this.regionList
        }
        ,
        mounted() {
            this.generateChart();
            this.updateChart();
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
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

    .visHidden {
        border: 0;
        clip: rect(0 0 0 0);
        height: 1px;
        margin: -1px;
        overflow: hidden;
        padding: 0;
        position: absolute;
        width: 1px;
    }

    .like-label {
        box-sizing: border-box;
        margin: 5px;
        padding: 5px;
        border: 1px solid gray;
        border-radius: 4px;
    }

    label {
        box-sizing: border-box;
        margin: 5px;
        padding: 5px;
        border: 1px solid gray;
        border-radius: 4px;
    }

    .noselect {
        -webkit-touch-callout: none; /* iOS Safari */
        -webkit-user-select: none; /* Safari */
        -khtml-user-select: none; /* Konqueror HTML */
        -moz-user-select: none; /* Firefox */
        -ms-user-select: none; /* Internet Explorer/Edge */
        user-select: none;
        /* Non-prefixed version, currently
                                         supported by Chrome and Opera */
    }

</style>
