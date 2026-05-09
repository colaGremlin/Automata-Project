<template>
  <section class="automata">
    <div class="automata__controls">
      <base-input
        type="text"
        v-model="regularExpression"
        labelText="Regular expression"
        placeholder="(a+b)*ab"
        class="automata__input"
        :error="validationError"
        :hint="hint"
      ></base-input>

      <base-select
        :options="options"
        v-model="type"
        label="Automaton"
        class="automata__select"
      ></base-select>
    </div>

    <automata-tester
      :automaton="testAutomaton"
      :alphabets="testAutomaton ? testAutomaton.alphabets : []"
      @update:activeStates="onActiveStates"
      @update:activeEdges="onActiveEdges"
    />

    <div class="automata__graph-pane">
      <transition name="fade">
        <automata-graph
          v-if="automaton"
          :automata="automaton"
          :activeStates="activeStates"
          :activeEdges="activeEdges"
        ></automata-graph>
        <div v-else class="automata__placeholder">
          <div class="automata__placeholder-card">
            <p class="automata__placeholder-title">{{ placeholderTitle }}</p>
            <p class="automata__placeholder-sub">{{ placeholderSub }}</p>
          </div>
        </div>
      </transition>
    </div>
  </section>
</template>

<script setup>
import { computed, ref } from "vue";
import { FiniteAutomata } from "@/utilities/finiteAutomata.js";
import AutomataGraph from "@/components/Automata/AutomataGraph.vue";
import AutomataTester from "@/components/Automata/AutomataTester.vue";

const regularExpression = ref("");
const type = ref("NFA");
const options = ["NFA", "DFA", "Min DFA"];
const activeStates = ref([]);
const activeEdges = ref([]);

// Strip whitespace and replace "" with λ (null string)
const cleanedExpression = computed(() =>
  regularExpression.value.replace(/\s/g, "").replace(/""/g, "λ")
);

const isValidExpression = computed(() => {
  if (!cleanedExpression.value) return false;
  return Boolean(
    new FiniteAutomata().validateRegularExpression(cleanedExpression.value)
  );
});

const validationError = computed(() => {
  if (!regularExpression.value) return "";
  return isValidExpression.value
    ? ""
    : "Invalid expression. Use letters, digits, +, *, and matching parentheses.";
});

const hint = computed(() => {
  if (regularExpression.value) return "";
  return 'Operators: + (or), * (zero or more), juxtaposition (concat). Use ( ) for grouping. Use "" for null string.';
});

const baseNFA = computed(() => {
  if (!isValidExpression.value) return null;
  return new FiniteAutomata().parseRegularExpression(cleanedExpression.value);
});

const automaton = computed(() => {
  if (!baseNFA.value) return null;
  if (type.value === "NFA") return baseNFA.value;
  if (type.value === "DFA") return baseNFA.value.toDFA();
  if (type.value === "Min DFA") return baseNFA.value.toMiniDFA();
  return null;
});

const testAutomaton = computed(() => automaton.value);

const placeholderTitle = computed(() => {
  if (!regularExpression.value) return "No expression yet";
  if (!isValidExpression.value) return "Invalid expression";
  return "Generating…";
});

const placeholderSub = computed(() => {
  if (!regularExpression.value)
    return "Enter a regular expression above to build the automaton.";
  if (!isValidExpression.value)
    return "Fix the syntax to render the diagram.";
  return "";
});

const onActiveStates = (states) => {
  activeStates.value = states;
};
const onActiveEdges = (edges) => {
  activeEdges.value = edges;
};
</script>

<style scoped lang="scss">
.automata {
  display: flex;
  flex-direction: column;
  gap: 1.2rem;

  &__controls {
    display: flex;
    align-items: flex-end;
    gap: 1.2rem;
    flex-wrap: wrap;
  }

  &__input {
    flex: 1;
    min-width: 28rem;
  }

  &__graph-pane {
    position: relative;
    min-height: min(72vh, 78rem);
    height: min(72vh, 78rem);
    border-radius: 1.2rem;
    overflow: hidden;
  }

  &__placeholder {
    width: 100%;
    height: 100%;
    background: transparent;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  &__placeholder-card {
    text-align: center;
    max-width: 36rem;
    padding: 2rem;
  }

  &__placeholder-title {
    color: var(--text-primary);
    font-size: 1.6rem;
    font-weight: 600;
    margin-bottom: 0.4rem;
  }

  &__placeholder-sub {
    color: var(--text-secondary);
    font-size: 1.3rem;
    line-height: 1.5;
  }
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
