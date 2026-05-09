<template>
  <div
    class="input"
    :class="{
      'input--error': !!error,
      'input--has-value': !!modelValue,
    }"
  >
    <label v-if="labelText" class="input__label">{{ labelText }}</label>
    <div class="input__field-wrap">
      <input
        :type="type"
        :class="['input__field', { 'input__field--mono': mono }]"
        :value="modelValue"
        :placeholder="placeholder"
        :spellcheck="false"
        autocomplete="off"
        @input="onInput"
      />
      <slot name="suffix"></slot>
    </div>
    <p v-if="error" class="input__error">{{ error }}</p>
    <p v-else-if="hint" class="input__hint">{{ hint }}</p>
  </div>
</template>

<script setup>
const emits = defineEmits(["update:modelValue"]);

defineProps({
  type: {
    type: String,
    default: "text",
    validator(value) {
      return [
        "date",
        "email",
        "text",
        "number",
        "search",
        "tel",
        "time",
        "url",
        "month",
      ].includes(value);
    },
  },
  labelText: { type: String, default: "" },
  modelValue: { type: String, default: "" },
  placeholder: { type: String, default: "" },
  mono: { type: Boolean, default: false },
  hint: { type: String, default: "" },
  error: { type: String, default: "" },
});

const onInput = (event) => {
  emits("update:modelValue", event.target.value);
};
</script>

<style lang="scss" scoped>
.input {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
  font-family: var(--font-ui);

  &__label {
    font-size: 1.1rem;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: var(--text-muted);
  }

  &__field-wrap {
    position: relative;
    display: flex;
    align-items: center;
  }

  &__field {
    width: 100%;
    height: 4.4rem;
    padding: 0 1.4rem;
    background: var(--bg-surface-2);
    border: 0.1rem solid var(--border);
    border-radius: 0.8rem;
    color: var(--text-primary);
    font-size: 1.5rem;
    font-family: var(--font-mono);
    font-weight: 500;
    transition: border-color 0.15s, background 0.15s, box-shadow 0.15s;

    &::placeholder {
      color: var(--text-muted);
      font-family: var(--font-mono);
    }

    &:hover {
      border-color: var(--border-strong);
    }

    &:focus {
      border-color: var(--accent);
      background: var(--bg-elevated);
      box-shadow: 0 0 0 0.3rem var(--accent-soft);
    }
  }

  &--error &__field {
    border-color: var(--danger);
    box-shadow: 0 0 0 0.3rem var(--danger-soft);
  }

  &__hint {
    font-size: 1.1rem;
    color: var(--text-muted);
  }

  &__error {
    font-size: 1.1rem;
    color: var(--danger);
    font-weight: 500;
  }
}
</style>
