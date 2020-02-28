<template>
  <pre
    ref="container"
    class="code-container language-js"
    :class="highlightLines.length > 0 && 'highlighting'"
  ><div><code
  v-for="(change, index) in html"
  :ref="change.id"
  class="list-item"
  :class="highlightLines.includes(index) && 'highlight'"
  :key="change.id"
  v-html="change.value + '\n'"
/></div></pre>
</template>

<script>
import Prism from "prismjs";
import { diffLines } from "diff";
import { tween } from "shifty";

const stripPx = str => Number.parseInt(str.substr(0, str.length - 2));

export default {
  props: {
    code: {
      type: Object
    }
  },

  data() {
    return {
      html: [],
      highlightLines: [],
      sequence: [],
      additive: false
    };
  },

  watch: {
    code(newVal, oldVal) {
      const diff = diffLines(oldVal.code, newVal.code);

      const toHtml = Prism.highlight(
        newVal.code,
        Prism.languages.javascript,
        "javascript"
      ).split("\n");
      const fromHtml = Prism.highlight(
        oldVal.code,
        Prism.languages.javascript,
        "javascript"
      ).split("\n");

      this.additive = toHtml.length > fromHtml.length;

      let toLine = 0;
      let fromLine = 0;
      for (let change of diff) {
        if (change.added) {
          change.html = toHtml.slice(toLine, toLine + change.count);
          toLine += change.count;
        } else if (change.removed) {
          change.html = fromHtml.slice(fromLine, fromLine + change.count);
          fromLine += change.count;
        } else {
          change.html = toHtml.slice(toLine, toLine + change.count);
          toLine += change.count;
          fromLine += change.count;
        }
      }

      const from = diff
        .reduce((prev, next) => {
          if (next.added) {
            return prev.concat(new Array(next.count).fill(""));
          } else {
            return prev.concat(next.html);
          }
        }, [])
        .map((value, index) => ({
          value,
          id: index + ""
        }))
        .filter(({ value }) => value !== "");

      const transition = diff
        .reduce((prev, next) => {
          if (next.added || next.removed) {
            return prev.concat(new Array(next.count).fill(""));
          } else {
            return prev.concat(next.html);
          }
        }, [])
        .map((value, index) => ({
          value,
          id: value === "" ? `${index}-change` : index + ""
        }));

      const to = diff
        .reduce((prev, next) => {
          if (next.removed) {
            return prev.concat(new Array(next.count).fill(""));
          } else {
            return prev.concat(next.html);
          }
        }, [])
        .map((value, index) => ({
          value,
          id: index + ""
        }))
        .filter(({ value }) => value !== "");

      this.sequence = [
        { code: from, highlight: oldVal.highlightLines },
        { code: transition, highlight: newVal.highlightLines },
        { code: to, highlight: newVal.highlightLines }
      ];
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

      const state = this.sequence.shift();
      this.performTransition(state);

      setTimeout(this.processSequence, 500);
    },

    async performTransition(state) {
      const { code, highlight } = state;
      const { added, removed, unchanged } = this.correlateIds(this.html, code);

      this.setupFlip(unchanged);
      await this.transition(removed);

      this.html = code;
      this.highlightLines = highlight;

      await this.$nextTick();

      this.executeFlip();
      this.transition(added, true);
    },

    animateContainer() {},

    transition(changes, reverse = false) {
      const promises = changes.map((change, index) => {
        const el = this.$refs[change.id][0];

        if (change.value === "") {
          return Promise.resolve();
        }

        let from = {
          opacity: 1,
          left: 0
        };
        let to = {
          opacity: 0,
          left: 600
        };

        if (reverse) {
          const swap = from;
          from = to;
          to = swap;
        }

        return tween({
          from,
          to,
          duration: 300,
          delay: index * 30,
          easing: "easeOutQuad",
          step: state => {
            // el.style.opacity = state.opacity + "";
            el.style.transform = `translateX(${state.left}px)`;
          }
        }).then(() => {
          el.style.opacity = "";
          el.style.transform = undefined;
        });
      });

      return Promise.all(promises);
    },

    setupFlip(changes) {
      const el = changes.map(change => this.$refs[change.id][0]);
      const fromPos = el.map(el => el.getBoundingClientRect());

      el.forEach(el => {
        el.style.transform = "";
        el.classList.remove("list-move");
      });

      this.flip = {
        el,
        fromPos,
        containerHeight: this.$refs.container.scrollHeight
      };
    },

    executeFlip() {
      const { el, fromPos } = this.flip;

      // Calculate offsets
      const toPos = el.map(el => el.getBoundingClientRect());
      const offset = fromPos.reduce((prev, next, index) => {
        prev.push({
          left: next.left - toPos[index].left,
          top: next.top - toPos[index].top
        });
        return prev;
      }, []);

      offset.forEach(({ left, top }, index) => {
        tween({
          from: { left, top },
          to: { left: 0, top: 0 },
          duration: 500,
          easing: "easeOutQuad",
          step: state => {
            el[index].style.transform = `translate(${state.left}px, ${
              state.top
            }px)`;
          }
        }).then(() => {
          el[index].style.transform = undefined;
        });
      });

      // Animate container
      const container = this.$refs.container;
      const { paddingTop, paddingBottom } = getComputedStyle(container);
      const childHeight = container.firstChild.clientHeight;
      const height = stripPx(paddingTop) + childHeight + stripPx(paddingBottom);

      tween({
        from: { height: this.flip.containerHeight },
        to: { height },
        duration: 500,
        easing: "easeOutQuad",
        step: state => {
          container.style.height = `${state.height}px`;
        }
      }).then(() => {
        container.style.height = undefined;
      });
    },

    correlateIds(from, to) {
      const added = [];
      const removed = [];
      const unchanged = [];

      for (const change of from) {
        const found = to.some(({ id }) => id === change.id);

        if (found) {
          unchanged.push(change);
        } else {
          removed.push(change);
        }
      }

      for (const change of to) {
        const found = from.some(({ id }) => id === change.id);

        if (!found) {
          added.push(change);
        }
      }

      return {
        added,
        removed,
        unchanged
      };
    }
  }
};
</script>

<style>
.transition-container {
  overflow: hidden;
}

.list-item {
  transition: opacity 0.5s ease-in-out;
}

.highlighting .list-item:not(.highlight) {
  opacity: 0.5;
}

.code-container {
  max-width: 600px;
  box-sizing: border-box;
  overflow: hidden !important;
}

.list-item {
  display: block;
}
</style>
