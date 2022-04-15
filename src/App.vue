<template>
  <div id="app">
    <PdfViewer src="/3.pdf"/>
  </div>
</template>

<script>
import PdfViewer from './components/PdfViewer.vue'

export default {
  name: 'App',
  components: {
    PdfViewer
  }
}
</script>

<style>
* {
  padding: 0;
  margin: 0;
}

html {
  height: 100%;
  width: 100%;
}

body {
  height: 100%;
  width: 100%;
  background-color: #808080;
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>
