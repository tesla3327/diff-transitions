<template>
  <div id="app">
    <button @click="toggle = !toggle">Toggle</button>
    <Code :code="toggle ? before : after"/>
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
      toggle: true,
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
