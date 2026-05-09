<template>
  <div class="select" :class="{ 'select--open': isOpen }" ref="rootRef">
    <button
      type="button"
      class="select__trigger"
      @click="toggle"
      :aria-expanded="isOpen"
    >
      <span v-if="label" class="select__label">{{ label }}</span>
      <span class="select__value">{{ modelValue }}</span>
      <svg
        class="select__chevron"
        viewBox="0 0 12 12"
        width="12"
        height="12"
        aria-hidden="true"
      >
        <path
          d="M2.5 4.5 L6 8 L9.5 4.5"
          fill="none"
          stroke="currentColor"
          stroke-width="1.6"
          stroke-linecap="round"
          stroke-linejoin="round"
        />
      </svg>
    </button>

    <ul v-show="isOpen" class="select__menu" role="listbox">
      <li
        v-for="option in options"
        :key="option"
        :class="[
          'select__option',
          { 'select__option--active': option === modelValue },
        ]"
        role="option"
        :aria-selected="option === modelValue"
        @click="select(option)"
      >
        {{ option }}
      </li>
    </ul>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";

const props = defineProps({
  options: { type: Array, default: () => [] },
  modelValue: { type: String, default: "" },
  label: { type: String, default: "" },
});

const emits = defineEmits(["update:modelValue"]);

const isOpen = ref(false);
const rootRef = ref(null);

const toggle = () => (isOpen.value = !isOpen.value);
const close = () => (isOpen.value = false);
const select = (option) => {
  emits("update:modelValue", option);
  close();
};

const onDocClick = (event) => {
  if (rootRef.value && !rootRef.value.contains(event.target)) close();
};
const onKey = (event) => {
  if (event.key === "Escape") close();
};

onMounted(() => {
  document.addEventListener("mousedown", onDocClick);
  document.addEventListener("keydown", onKey);
});
onBeforeUnmount(() => {
  document.removeEventListener("mousedown", onDocClick);
  document.removeEventListener("keydown", onKey);
});
</script>

<style scoped lang="scss">
.select {
  position: relative;
  display: inline-block;
  font-family: var(--font-ui);

  &__trigger {
    display: flex;
    align-items: center;
    gap: 1rem;
    background: var(--bg-surface-2);
    border: 0.1rem solid var(--border);
    border-radius: 0.8rem;
    padding: 0 1.2rem;
    height: 4.4rem;
    color: var(--text-primary);
    font-size: 1.4rem;
    font-weight: 500;
    cursor: pointer;
    transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;

    &:hover {
      border-color: var(--border-strong);
      background: var(--bg-elevated);
    }
  }

  &--open &__trigger {
    border-color: var(--accent);
    box-shadow: 0 0 0 0.3rem var(--accent-soft);
  }

  &__label {
    color: var(--text-muted);
    font-size: 1.1rem;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }

  &__value {
    color: var(--text-primary);
    min-width: 6rem;
    text-align: left;
    font-weight: 600;
  }

  &__chevron {
    color: var(--text-secondary);
    transition: transform 0.2s ease;
  }
  &--open &__chevron {
    transform: rotate(180deg);
    color: var(--accent);
  }

  &__menu {
    position: absolute;
    top: calc(100% + 0.4rem);
    right: 0;
    min-width: 100%;
    background: var(--bg-elevated);
    border: 0.1rem solid var(--border-strong);
    border-radius: 0.8rem;
    padding: 0.4rem;
    z-index: 100;
    box-shadow: var(--shadow-lg);
  }

  &__option {
    padding: 0.8rem 1.2rem;
    font-size: 1.4rem;
    color: var(--text-secondary);
    border-radius: 0.5rem;
    cursor: pointer;
    transition: background 0.1s, color 0.1s;
    white-space: nowrap;
    font-weight: 500;

    &:hover {
      background: var(--bg-surface-2);
      color: var(--text-primary);
    }

    &--active {
      background: var(--accent-soft);
      color: var(--accent-hover);
    }
  }
}
</style>
