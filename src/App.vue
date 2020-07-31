<template>
  <div id="app">
    <Container :key="reloadKey" @reload="reloadKey += 1" />
  </div>
</template>
<script>
import Container from './components/Container.vue'
export default {
  name: 'App',
  components: {
    Container
  },
  data: () => ({
    reloadKey: 0
  })
}
</script>
<style>
@import url('https://fonts.googleapis.com/css2?family=Martel+Sans&display=swap');
#app {
  font-family: 'Martel Sans', sans-serif;
}
a {
  color: #303;
}
</style>
