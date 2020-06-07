<template>
  <div id="app">
    <Chart :data=loadData />
  </div>
</template>

<script>
import Chart from './components/Chart.vue'
const csvPath = require("./assets/carsDE.csv")

export default {
  name: 'App',
  components: {
    Chart
  },
  data: function() {
    return {
      loadData: csvPath
    };
  },
  mounted() {
    console.log("App loaded");
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 30px;
  min-width:900px;
}
</style>
