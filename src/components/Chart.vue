<template>
    <div id="chartComponent">
        <h1>{{ msg }}</h1>
    </div>
</template>

<script>
    import * as d3 from "d3"

    export default {
        name: 'Chart',
        props: {
            data: Array
        },
        computed: {
            chartData: function(){
                return this.data
                    .map((car, i) => {return {index: i, manufacturer: car.Hersteller, region: car.Herkunft, x: Number(car.Gewicht), y: Number(car.Verbrauch)}})
                    .filter(entry => !isNaN(entry.x) && !isNaN(entry.y))
                }
        },
        data() {
            return {
                msg: "🚗 🚙 🚓 🚕 🚙 🚓 🚗 🚕",
                height: 600,
                width: 800,
                margin: {top: 25, right: 20, bottom: 35, left: 40}
            }
        },
        methods: {
            generateChart() {
                const color = d3.scaleOrdinal().range([...Array(30).keys()].map(i => `hsl(${i * 12}, 100%, 60%)`));
                const shape = d3.scaleOrdinal(this.chartData.map(d => d.category), d3.symbols.map(s => d3.symbol().type(s)()));

                const x = d3.scaleLinear()
                    .domain(d3.extent(this.chartData, d => d.x)).nice()
                    .range([this.margin.left, this.width - this.margin.right])
                const y = d3.scaleLinear()
                    .domain(d3.extent(this.chartData, d => d.y)).nice()
                    .range([this.height - this.margin.bottom, this.margin.top]);

                const xAxis = g => g
                    .attr("transform", `translate(0,${this.height - this.margin.bottom})`)
                    .call(d3.axisBottom(x).ticks(this.width / 80))
                    .call(g => g.select(".domain").remove())
                    .call(g => g.append("text")
                        .attr("x", this.width)
                        .attr("y", this.margin.bottom - 4)
                        .attr("fill", "currentColor")
                        .attr("text-anchor", "end")
                        .text("X-Achse"))

                const yAxis = g => g
                    .attr("transform", `translate(${this.margin.left},0)`)
                    .call(d3.axisLeft(y))
                    .call(g => g.select(".domain").remove())
                    .call(g => g.append("text")
                        .attr("x", -this.margin.left)
                        .attr("y", 10)
                        .attr("fill", "currentColor")
                        .attr("text-anchor", "start")
                        .text("Y-Achse"))

                const grid = g => g
                    .attr("stroke", "currentColor")
                    .attr("stroke-opacity", 0.1)
                    .call(g => g.append("g")
                        .selectAll("line")
                        .data(x.ticks())
                        .join("line")
                        .attr("x1", d => 0.5 + x(d))
                        .attr("x2", d => 0.5 + x(d))
                        .attr("y1", this.margin.top)
                        .attr("y2", this.height - this.margin.bottom))
                    .call(g => g.append("g")
                        .selectAll("line")
                        .data(y.ticks())
                        .join("line")
                        .attr("y1", d => 0.5 + y(d))
                        .attr("y2", d => 0.5 + y(d))
                        .attr("x1", this.margin.left)
                        .attr("x2", this.width - this.margin.right));

                const componentDiv = d3.select("#chartComponent")
                const svg = componentDiv
                    .append("svg")
                    .attr("viewBox", [0, 0, this.width, this.height]);

                const tooltipRect = svg.append("rect")
                    .attr("fill", "white")
                    .attr("stroke", "black")
                const tooltip = svg.append("text")

                svg.append("g")
                    .call(xAxis);

                svg.append("g")
                    .call(yAxis);

                svg.append("g")
                    .call(grid);

                svg.append("g")
                    .attr("stroke-width", 0.5)
                    .attr("font-family", "sans-serif")
                    .attr("font-size", 50)
                    .selectAll("path")
                    .data(this.chartData)
                    .join("path")
                    .on("mouseover", d => {
                        tooltip.attr("visibility", "visible")
                        tooltipRect.attr("visibility", "visible")
                        tooltip.text(this.data[d.index].Model);
                        const bb = tooltip.node().getBBox();
                        tooltip.attr("x", (x(d.x) - bb.width/2)).attr("y", (y(d.y) - bb.height/2));
                        tooltipRect.attr("x", (x(d.x) - bb.width*0.5)).attr("y", (y(d.y) - bb.height*1.25)).attr("width", bb.width).attr("height", bb.height);
                        d3.select(d3.event.target).attr("stroke", "black").raise()
                    })
                    .on("mouseout", () => {
                        tooltip.attr("visibility", "hidden")
                        tooltipRect.attr("visibility", "hidden")
                        d3.select(d3.event.target).attr("stroke", "none")
                    })
                    .attr("transform", d => `translate(${x(d.x)},${y(d.y)})`)
                    .attr("fill", d => color(d.manufacturer))
                    .attr("d", d => shape(d.region));

                tooltipRect.raise();
                tooltip.raise();
            }
        },
        mounted() {
            this.generateChart()
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
