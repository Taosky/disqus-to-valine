<template>
  <div id="app">
    <Transfer/>
  </div>
</template>

<script>
import Transfer from './components/Transfer.vue'

export default {
  name: 'app',
  components: {
      Transfer
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 10px;
}
</style>
