<template>
  <pre ref="code" class="code-container language-js"><transition-group name="list" tag="div"><code
  :ref="change.id"
  class="list-item"
  v-for="change in html"
  :key="change.id"
  v-html="change.value + '\n'"
/></transition-group></pre>
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
      html: [],
      sequence: []
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

      this.sequence.push(old, from, to);
    },

    sequence() {
      if (this.executingSequence) return;
      this.executingSequence = true;
      this.processSequence();
    }
  },

  methods: {
    processSequence() {
      if (this.sequence.length === 0) {
        this.executingSequence = false;
        return;
      }

      const html = this.sequence.shift();
      this.updateHtml(html);

      setTimeout(this.processSequence, 1000);
    },

    updateHtml(html) {
      const el = this.$refs.code;
      const { transition } = el.style;
      el.style.transition = "";

      // First transition out existing HTML
      this.transitionIds = this.html
        .filter(change => !html.find(({ id }) => change.id === id))
        .map(change => change.id);

      if (this.transitionIds.length > 0) {
        this.addClass("transition-leave-from");
      }

      requestAnimationFrame(() => {
        if (this.transitionIds.length > 0) {
          this.removeClass("transition-leave-from");
          this.addClass("transition-leave-to");

          const transEl = this.$refs[this.transitionIds[0]][0];
          let setupTransitionEnd = false;

          transEl.addEventListener("transitionstart", () => {
            transEl.addEventListener(
              "transitionend",
              this.handleCodeTransitionEnd.bind(this, html, transition)
            );
            setupTransitionEnd = true;
          });

          requestAnimationFrame(() => {
            // No CSS transition is being run, so just keep going
            if (!setupTransitionEnd) {
              this.transitionTo(html, transition);
            }
          });
        } else {
          this.transitionTo(html, transition);
        }
      });
    },

    handleCodeTransitionEnd(html, transition) {
      console.log("transition end");
      this.removeClass("transition-leave-to");

      const el = this.$refs[this.transitionIds[0]][0];
      if (el) {
        el.removeEventListener("transitionend", this.handleCodeTransitionEnd);
      }

      this.transitionTo(html, transition);
    },

    transitionTo(html, transition) {
      const el = this.$refs.code;

      // Wait until the CSS properties have updated
      requestAnimationFrame(() => {
        el.style.height = `${el.scrollHeight}px`;
        el.style.transition = transition;

        this.html = html;

        this.removeTransitionLines();

        this.$nextTick(() => {
          if (this.transitionIds.length > 0) {
            // Remove .list-move class from all to prevent
            // unnecessary movement while new lines are transitioned in
            this.$el.querySelectorAll(".list-move").forEach(moveEl => {
              moveEl.classList.remove("list-move");
            });
          }

          const { paddingTop, paddingBottom } = getComputedStyle(el);
          const childHeight = el.firstChild.clientHeight;
          const height =
            stripPx(paddingTop) + childHeight + stripPx(paddingBottom);

          el.style.height = `${height}px`;
        });
      });
    },

    removeTransitionLines() {
      this.transitionIds.forEach(id => {
        const el = this.$refs[id][0];
        if (el) {
          el.remove();
        }
      });
    },

    handleTransitionEnd() {
      this.$refs.code.style.height = undefined;
    },

    addClass(className) {
      this.transitionIds.forEach(id => {
        const el = this.$refs[id][0];
        if (el) {
          el.classList.add(className);
        }
      });
    },

    removeClass(className) {
      this.transitionIds.forEach(id => {
        const el = this.$refs[id][0];
        if (el) {
          el.classList.remove(className);
        }
      });
    }
  }
};
</script>

<style>
.code-container {
  transition: height 1s ease-in-out;
  box-sizing: border-box;
}

.list-item {
  display: block;
}

.list-enter-active,
.list-leave-active {
  transition: all 1s ease-in-out;
}

.list-enter,
.list-leave-to {
  opacity: 0;
}

.list-move {
  transition: all 1s ease-in-out;
}

.list-leave-active {
}

.transition-leave-to,
.transition-leave-from {
  transition: background-color 1s ease-in-out;
}

.transition-leave-from {
  background-color: red;
}

.transition-leave-to {
}
</style>
