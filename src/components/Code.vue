<template>
  <pre ref="container" class="code-container language-js"><div><code
  :ref="change.id"
  class="list-item"
  v-for="change in html"
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
      type: String
    }
  },

  data() {
    return {
      html: [],
      sequence: [],
      additive: false
    };
  },

  watch: {
    code(newVal, oldVal) {
      const diff = diffLines(oldVal, newVal);

      const toHtml = Prism.highlight(
        newVal,
        Prism.languages.javascript,
        "javascript"
      ).split("\n");
      const fromHtml = Prism.highlight(
        oldVal,
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

      this.sequence = [from, transition, to];
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
      this.performTransition(html);

      setTimeout(this.processSequence, 1000);
    },

    async performTransition(newHtml) {
      const { added, removed, unchanged } = this.correlateIds(
        this.html,
        newHtml
      );

      this.setupFlip(unchanged);
      await this.leave(removed);

      this.html = newHtml;

      await this.$nextTick();

      this.executeFlip();
      this.enter(added);
    },

    animateContainer() {},

    leave(removed) {
      // Transition out the removed lines
      const promises = removed.map(change => {
        const el = this.$refs[change.id][0];

        if (change.value === "") {
          return Promise.resolve();
        }

        return tween({
          from: { opacity: 1 },
          to: { opacity: 0 },
          duration: 500,
          easing: "easeOutQuad",
          step: state => {
            el.style.opacity = state.opacity + "";
          }
        });
      });

      return Promise.all(promises);
    },

    enter(added) {
      // Transition in the added lines
      const promises = added.map(change => {
        const el = this.$refs[change.id][0];

        if (change.value === "") {
          return Promise.resolve();
        }

        return tween({
          from: { opacity: 0 },
          to: { opacity: 1 },
          duration: 500,
          easing: "easeOutQuad",
          step: state => {
            el.style.opacity = state.opacity + "";
          }
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

.code-container {
  transition: height 1s ease-in-out;
  box-sizing: border-box;
}

.list-item {
  display: block;
}
</style>
