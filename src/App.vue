<template>
  <div id="app">
    <CoviBoard welcome="Welcome to vue-d3-pand"/>
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
  html {
    background: black;
  }

  #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
    background: black;
  }
</style>
