<template>
  <pre ref="code" class="language-js"><transition-group name="list-complete" mode="out-in" tag="div"><code class="list-item" v-for="change in html" :key="change.id" v-html="change.value + '\n'"/></transition-group></pre>
</template>

<script>
import Prism from "prismjs";
import { diffLines } from "diff";

export default {
  props: {
    code: {
      type: String
    }
  },

  data() {
    return {
      html: ""
    };
  },

  computed: {
    // html() {
    //   return Prism.highlight(
    //     this.code,
    //     Prism.languages.javascript,
    //     "javascript"
    //   );
    // }
  },

  watch: {
    code(newVal, oldVal) {
      const diff = diffLines(oldVal, newVal);
      const html = Prism.highlight(
        newVal,
        Prism.languages.javascript,
        "javascript"
      ).split("\n");

      for (let change of diff) {
        change.html = html.splice(0, change.count);
      }

      const old = diff
        .reduce((prev, next) => {
          if (!next.added) {
            return prev.concat(next.html);
          } else {
            return prev.concat(new Array(next.count).fill(""));
          }
        }, [])
        .map((value, index) => ({
          value,
          id: index
        }))
        .filter(({ value }) => value !== "");

      const from = diff
        .reduce((prev, next) => {
          if (!next.added) {
            return prev.concat(next.html);
          } else {
            return prev.concat(new Array(next.count).fill(""));
          }
        }, [])
        .map((value, index) => ({
          value,
          id: value === "" ? `${index}-change` : index
        }));

      const to = diff
        .reduce((prev, next) => {
          return prev.concat(next.html);
        }, [])
        .map((value, index) => ({
          value,
          id: index
        }));

      this.updateHtml(old);
      setTimeout(() => {
        this.updateHtml(from);

        setTimeout(() => {
          this.updateHtml(to);
        }, 1000);
      }, 1000);
    }
  },

  methods: {
    updateHtml(html) {
      this.html = html;
    }
  }
};
</script>

<style>
.list-item {
  display: block;
  transition: all 0.7s ease-in-out;
}
.list-complete-enter, .list-complete-leave-to
/* .list-complete-leave-active below version 2.1.8 */ {
  opacity: 0;
  transform: translateX(30px);
}
.list-complete-move {
  opacity: 1;
  transition: transform 0.7s ease-in-out;
}
.list-complete-leave-active {
  display: none;
}
</style>