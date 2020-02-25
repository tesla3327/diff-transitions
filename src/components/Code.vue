<template>
  <pre ref="code" class="code-container language-js"><transition-group name="list-complete" tag="div"><code class="list-item" v-for="change in html" :key="change.id" v-html="change.value + '\n'"/></transition-group></pre>
</template>

<script>
import Prism from "prismjs";
import { diffLines } from "diff";

const stripPx = str => Number.parseInt(str.substr(0, str.length - 2));

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

  mount() {
    this.$refs.code.addEventListener("transitionend", this.handleTransitionEnd);
  },

  beforeDestroy() {
    this.$refs.code.removeEventListener(
      "transitionend",
      this.handleTransitionEnd
    );
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

          setTimeout(() => {
            this.updateHtml(old);
          }, 1000);
        }, 1000);
      }, 1000);
    }
  },

  methods: {
    updateHtml(html) {
      const el = this.$refs.code;

      const { transition } = el.style;
      el.style.transition = "";

      // Wait until the CSS properties have updated
      requestAnimationFrame(() => {
        el.style.height = `${el.scrollHeight}px`;
        el.style.transition = transition;

        this.html = html;

        this.$nextTick(() => {
          const { paddingTop, paddingBottom } = getComputedStyle(el);
          const childHeight = el.firstChild.clientHeight;
          const height =
            stripPx(paddingTop) + childHeight + stripPx(paddingBottom);

          el.style.height = `${height}px`;
        });
      });
    },

    handleTransitionEnd() {
      this.$refs.code.style.height = undefined;
    }
  }
};
</script>

<style>
.code-container {
  transition: height 0.7s ease-in-out;
  box-sizing: border-box;
}

.list-item {
  display: block;
  transition: opacity 0.7s ease-in-out;
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
