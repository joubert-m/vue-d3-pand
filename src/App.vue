<template>
  <div id="app">
    <CoviBoard msg="Welcome to vue-d3-pand"/>
  </div>
</template>

<script>
import CoviBoard from './components/CovidBoard.vue'

export default {
  name: 'App',
  components: {
    CoviBoard
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
  margin-top: 60px;
}
</style>
