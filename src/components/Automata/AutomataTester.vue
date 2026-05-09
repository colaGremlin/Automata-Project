<template>
  <section class="tester">
    <div class="tester__row">
      <div class="tester__input-wrap">
        <span class="tester__chip">test</span>
        <div class="tester__field" :class="fieldClass">
          <input
            ref="inputEl"
            type="text"
            class="tester__input"
            :value="testString"
            :placeholder="placeholder"
            spellcheck="false"
            autocomplete="off"
            @input="onInput"
            @keydown.enter.prevent="step"
          />
          <div class="tester__overlay" aria-hidden="true">
            <span
              v-for="(ch, i) in chars"
              :key="i"
              :class="['tester__char', charClass(i)]"
              >{{ ch }}</span
            >
          </div>
        </div>
      </div>

      <div class="tester__status" :class="statusClass">
        <span class="tester__status-dot"></span>
        <span class="tester__status-text">{{ statusText }}</span>
      </div>

      <div class="tester__controls">
        <button
          class="tester__btn"
          type="button"
          @click="reset"
          :disabled="!canTest || (currentIndex === 0 && !isPlaying)"
          title="Reset"
        >
          <svg viewBox="0 0 14 14" width="12" height="12" aria-hidden="true">
            <path
              d="M3 2 L3 12 M11 2 L4 7 L11 12 Z"
              fill="currentColor"
              stroke="currentColor"
              stroke-width="0.6"
              stroke-linejoin="round"
            />
          </svg>
        </button>

        <button
          class="tester__btn tester__btn--primary"
          type="button"
          @click="togglePlay"
          :disabled="!canTest"
          :title="isPlaying ? 'Pause' : 'Play'"
        >
          <svg
            v-if="!isPlaying"
            viewBox="0 0 14 14"
            width="12"
            height="12"
            aria-hidden="true"
          >
            <path d="M4 2 L11.5 7 L4 12 Z" fill="currentColor" />
          </svg>
          <svg v-else viewBox="0 0 14 14" width="12" height="12" aria-hidden="true">
            <rect x="3.5" y="2.5" width="2.6" height="9" fill="currentColor" rx="0.5" />
            <rect x="7.9" y="2.5" width="2.6" height="9" fill="currentColor" rx="0.5" />
          </svg>
        </button>

        <button
          class="tester__btn"
          type="button"
          @click="step"
          :disabled="!canTest || isFinished"
          title="Step forward"
        >
          <svg viewBox="0 0 14 14" width="12" height="12" aria-hidden="true">
            <path d="M3 2 L9 7 L3 12 Z" fill="currentColor" />
            <rect x="9.6" y="2.5" width="1.6" height="9" fill="currentColor" rx="0.4" />
          </svg>
        </button>

        <div class="tester__speed">
          <label class="tester__speed-label">speed</label>
          <select v-model.number="speed" class="tester__speed-select">
            <option :value="0.5">0.5x</option>
            <option :value="1">1x</option>
            <option :value="2">2x</option>
            <option :value="4">4x</option>
          </select>
        </div>
      </div>
    </div>
  </section>
</template>

<script setup>
import { ref, computed, watch, onBeforeUnmount } from "vue";

const props = defineProps({
  automaton: { type: Object, default: null },
  alphabets: { type: Array, default: () => [] },
});

const emits = defineEmits(["update:activeStates", "update:activeEdges"]);

const testString = ref("");
const currentIndex = ref(0);
const activeStates = ref([]);
const activeEdges = ref([]);
const rejected = ref(false);
const isPlaying = ref(false);
const speed = ref(1);
const inputEl = ref(null);

let timer = null;

// Detect "" as null/empty string test
const isNullStringTest = computed(() => testString.value === '""');

const chars = computed(() => {
  if (isNullStringTest.value) return ["λ"];
  return (testString.value || "").split("");
});

const canTest = computed(
  () => !!props.automaton && testString.value.length > 0
);

const isFinished = computed(
  () =>
    rejected.value ||
    isNullStringTest.value ||
    (testString.value.length > 0 &&
      currentIndex.value >= testString.value.length)
);

const accepted = computed(() => {
  if (!props.automaton) return false;
  if (rejected.value) return false;
  if (isNullStringTest.value) {
    return activeStates.value.some((s) =>
      props.automaton.finalStates.includes(s)
    );
  }
  if (currentIndex.value < testString.value.length) return false;
  return activeStates.value.some((s) =>
    props.automaton.finalStates.includes(s)
  );
});

const placeholder = computed(() => {
  if (!props.automaton) return "Enter a regular expression first…";
  if (props.alphabets.length) {
    const symbols = props.alphabets.filter((a) => a !== "λ").join(", ");
    return `Type a string over { ${symbols} }  ·  use \"\" for null string`;
  }
  return 'Type a string to test…  ·  use "" for null string';
});

const fieldClass = computed(() => ({
  "tester__field--accepted":
    isFinished.value && !rejected.value && accepted.value,
  "tester__field--rejected":
    isFinished.value && (rejected.value || !accepted.value),
  "tester__field--disabled": !props.automaton,
}));

const statusText = computed(() => {
  if (!props.automaton) return "Awaiting expression";
  if (testString.value.length === 0) return "Idle";
  if (rejected.value) return "Rejected";
  if (isNullStringTest.value) return accepted.value ? "Accepted" : "Rejected";
  if (currentIndex.value === 0) return "Ready";
  if (currentIndex.value < testString.value.length) return "Running";
  return accepted.value ? "Accepted" : "Rejected";
});

const statusClass = computed(() => {
  if (!props.automaton || testString.value.length === 0)
    return "tester__status--idle";
  if (rejected.value || (isFinished.value && !accepted.value))
    return "tester__status--rejected";
  if (isFinished.value && accepted.value) return "tester__status--accepted";
  if (currentIndex.value === 0) return "tester__status--ready";
  return "tester__status--running";
});

const charClass = (i) => {
  if (isNullStringTest.value) {
    return accepted.value ? "tester__char--consumed" : "tester__char--rejected-cur";
  }
  if (rejected.value && i === currentIndex.value)
    return "tester__char--rejected-cur";
  if (i < currentIndex.value) {
    if (rejected.value) return "tester__char--past";
    return isFinished.value && !accepted.value
      ? "tester__char--past"
      : "tester__char--consumed";
  }
  if (i === currentIndex.value && !isFinished.value)
    return "tester__char--current";
  return "tester__char--future";
};

// λ-closure that also returns every λ-edge it walked, so the graph can light them up.
const lambdaClosureWithEdges = (automaton, seedStates) => {
  const closure = new Set(seedStates);
  const edges = [];
  const stack = [...seedStates];
  while (stack.length) {
    const s = stack.pop();
    if (s == null || s >= automaton.states.length) continue;
    for (const t of automaton.states[s]) {
      if (t.value === "λ") {
        edges.push({ from: s, to: t.to, value: "λ" });
        if (!closure.has(t.to)) {
          closure.add(t.to);
          stack.push(t.to);
        }
      }
    }
  }
  return { closure: [...closure], edges };
};

const startStates = (automaton) => {
  if (!automaton) return { closure: [], edges: [] };
  return lambdaClosureWithEdges(automaton, [0]);
};

const stopTimer = () => {
  if (timer) {
    clearTimeout(timer);
    timer = null;
  }
};

const reset = () => {
  stopTimer();
  isPlaying.value = false;
  rejected.value = false;
  currentIndex.value = 0;
  if (!props.automaton || testString.value.length === 0) {
    activeStates.value = [];
    activeEdges.value = [];
    return;
  }
  const { closure, edges } = startStates(props.automaton);
  activeStates.value = closure;
  activeEdges.value = edges;
};

const step = () => {
  if (!props.automaton || isFinished.value) return;
  if (currentIndex.value >= testString.value.length) return;

  const ch = testString.value[currentIndex.value];
  const directEdges = [];
  const direct = new Set();
  for (const s of activeStates.value) {
    if (s == null || s >= props.automaton.states.length) continue;
    for (const t of props.automaton.states[s]) {
      if (t.value === ch) {
        direct.add(t.to);
        directEdges.push({ from: s, to: t.to, value: ch });
      }
    }
  }
  if (direct.size === 0) {
    rejected.value = true;
    activeEdges.value = [];
    stopTimer();
    isPlaying.value = false;
    return;
  }
  const { closure, edges: lambdaEdges } = lambdaClosureWithEdges(
    props.automaton,
    [...direct]
  );
  activeStates.value = closure;
  activeEdges.value = [...directEdges, ...lambdaEdges];
  currentIndex.value += 1;
  if (currentIndex.value >= testString.value.length) {
    stopTimer();
    isPlaying.value = false;
  }
};

// Slower base interval so 1x feels like a deliberate walk.
const intervalAtSpeed = () => Math.max(120, 1100 / speed.value);

const playLoop = () => {
  if (!isPlaying.value) return;
  step();
  if (isPlaying.value && !isFinished.value) {
    timer = setTimeout(playLoop, intervalAtSpeed());
  } else {
    isPlaying.value = false;
    stopTimer();
  }
};

const togglePlay = () => {
  if (!canTest.value) return;
  if (isFinished.value) {
    reset();
  }
  isPlaying.value = !isPlaying.value;
  if (isPlaying.value) {
    playLoop();
  } else {
    stopTimer();
  }
};

const onInput = (event) => {
  testString.value = event.target.value || "";
  reset();
};

watch(
  () => props.automaton,
  () => {
    reset();
  }
);

watch(
  activeStates,
  (val) => emits("update:activeStates", val),
  { deep: true }
);
watch(
  activeEdges,
  (val) => emits("update:activeEdges", val),
  { deep: true }
);

onBeforeUnmount(stopTimer);
</script>

<style scoped lang="scss">
.tester {
  background: var(--bg-surface);
  border: 0.1rem solid var(--border);
  border-radius: 1.2rem;
  padding: 1.2rem 1.4rem;
  font-family: var(--font-ui);

  &__row {
    display: flex;
    align-items: center;
    gap: 1.2rem;
    flex-wrap: wrap;
  }

  &__input-wrap {
    flex: 1;
    min-width: 24rem;
    display: flex;
    align-items: center;
    gap: 0.8rem;
  }

  &__chip {
    font-size: 1rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.1em;
    color: var(--text-muted);
    padding: 0.3rem 0.6rem;
    background: var(--bg-surface-2);
    border-radius: 0.4rem;
    flex-shrink: 0;
  }

  &__field {
    position: relative;
    flex: 1;
    height: 4rem;
    background: var(--bg-surface-2);
    border: 0.1rem solid var(--border);
    border-radius: 0.6rem;
    transition: border-color 0.15s, background 0.15s;
    overflow: hidden;

    &:focus-within {
      border-color: var(--accent);
      background: var(--bg-elevated);
      box-shadow: 0 0 0 0.3rem var(--accent-soft);
    }

    &--accepted {
      border-color: var(--success);
      background: var(--success-soft);
    }
    &--rejected {
      border-color: var(--danger);
      background: var(--danger-soft);
    }
    &--disabled {
      opacity: 0.85;
    }
  }

  &__input {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    padding: 0 1.2rem;
    background: transparent;
    color: transparent;
    caret-color: var(--accent);
    font-family: var(--font-mono);
    font-size: 1.5rem;
    font-weight: 500;
    z-index: 1;

    &::placeholder {
      color: var(--text-muted);
      font-family: var(--font-mono);
      opacity: 1;
    }
    &::-moz-placeholder {
      color: var(--text-muted);
      opacity: 1;
    }
    &::-webkit-input-placeholder {
      color: var(--text-muted);
      opacity: 1;
    }

    &::selection {
      background: var(--accent-soft);
      color: var(--text-primary);
    }
  }

  &__overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    padding: 0 1.2rem;
    pointer-events: none;
    font-family: var(--font-mono);
    font-size: 1.5rem;
    font-weight: 500;
    line-height: 1;
    overflow: hidden;
    white-space: pre;
    z-index: 2;
  }

  &__char {
    transition: color 0.15s, background 0.15s;
    border-radius: 0.3rem;
    padding: 0.1rem 0;

    &--future {
      color: var(--text-secondary);
    }
    &--consumed {
      color: var(--accent);
    }
    &--past {
      color: var(--text-muted);
    }
    &--current {
      color: var(--bg-primary);
      background: var(--warning);
      box-shadow: 0 0 0 0.2rem var(--warning);
      font-weight: 700;
    }
    &--rejected-cur {
      color: #fff;
      background: var(--danger);
      box-shadow: 0 0 0 0.2rem var(--danger);
      font-weight: 700;
    }
  }

  &__status {
    display: inline-flex;
    align-items: center;
    gap: 0.6rem;
    font-size: 1.3rem;
    font-weight: 700;
    padding: 0.7rem 1.2rem;
    border-radius: 0.6rem;
    border: 0.15rem solid currentColor;
    text-transform: uppercase;
    letter-spacing: 0.06em;
    min-width: 11.5rem;
    justify-content: center;
    flex-shrink: 0;

    &-dot {
      width: 0.75rem;
      height: 0.75rem;
      border-radius: 50%;
      background: currentColor;
      box-shadow: 0 0 0.6rem currentColor;
    }

    &--idle {
      color: var(--text-muted);
      background: var(--bg-surface-2);
      border-color: var(--border);
    }
    &--ready {
      color: var(--accent);
      background: var(--accent-soft);
    }
    &--running {
      color: var(--warning);
      background: var(--warning-soft);
    }
    &--accepted {
      color: var(--success);
      background: var(--success-soft);
    }
    &--rejected {
      color: var(--danger);
      background: var(--danger-soft);
    }
  }

  &__controls {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    flex-shrink: 0;
  }

  &__btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 3.2rem;
    height: 3.2rem;
    border-radius: 0.6rem;
    background: var(--bg-surface-2);
    color: var(--text-secondary);
    border: 0.1rem solid var(--border);
    transition: all 0.15s;

    &:hover:not(:disabled) {
      color: var(--text-primary);
      border-color: var(--border-strong);
      background: var(--bg-elevated);
    }

    &:disabled {
      opacity: 0.4;
      cursor: not-allowed;
    }

    &--primary {
      background: var(--accent);
      color: var(--bg-primary);
      border-color: var(--accent);

      &:hover:not(:disabled) {
        background: var(--accent-hover);
        border-color: var(--accent-hover);
        color: var(--bg-primary);
      }
    }
  }

  &__speed {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    margin-left: 0.4rem;
  }

  &__speed-label {
    font-size: 1rem;
    color: var(--text-muted);
    text-transform: uppercase;
    letter-spacing: 0.08em;
    font-weight: 500;
  }

  &__speed-select {
    background: var(--bg-surface-2);
    border: 0.1rem solid var(--border);
    border-radius: 0.4rem;
    padding: 0.3rem 0.6rem;
    color: var(--text-primary);
    font-size: 1.2rem;
    font-family: var(--font-mono);
    font-weight: 500;
    cursor: pointer;
    -webkit-appearance: menulist;
    appearance: menulist;

    &:hover {
      border-color: var(--border-strong);
    }
  }
}
</style>
