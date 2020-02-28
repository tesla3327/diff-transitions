<template>
  <div id="app">
    <button @click="toggle = !toggle">Toggle</button>
    <Code :code="currentCode"/>
  </div>
</template>

<script>
import Code from "./components/Code";
import "./theme.css";

export default {
  name: "App",
  components: {
    Code
  },
  data() {
    return {
      toggle: false,
      beforeLines: [1],
      afterLines: [2, 3, 4],
      before: `export default () => {
  return 'this is cool' + 3;
};`,
      after: `export default () => {
  console.log('hello world');
  if ('green' == 'green') {
    return 'blue';
  }
  return 'this is cool' + 3;
};`
    };
  },

  computed: {
    currentCode() {
      return this.toggle
        ? { code: this.before, highlightLines: this.beforeLines }
        : { code: this.after, highlightLines: this.afterLines };
    }
  }
};
</script>

<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
