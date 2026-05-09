<template>
  <main class="main">
    <header class="main__header">
      <h1 class="main__title">
        Regex<span class="main__title-accent">To</span>FA
      </h1>
      <p class="main__subtitle">
        Convert a regular expression into NFA, DFA, or minimised DFA &mdash;
        then test strings against it
      </p>
    </header>

    <div class="main__content">
      <automata-box></automata-box>
    </div>
  </main>
</template>

<script setup>
import AutomataBox from "@/components/Automata/AutomataBox.vue";
</script>

<style scoped lang="scss">
.main {
  width: 100%;
  min-height: 100vh;
  padding: 2.4rem 3.2rem 3.2rem 3.2rem;
  display: flex;
  flex-direction: column;
  gap: 1.6rem;
  background: var(--bg-primary);

  &__header {
    flex-shrink: 0;
  }

  &__title {
    font-size: 2.8rem;
    font-weight: 700;
    letter-spacing: -0.02em;
    color: var(--text-primary);
    line-height: 1.1;
  }

  &__title-accent {
    color: var(--accent);
  }

  &__subtitle {
    font-size: 1.3rem;
    color: var(--text-secondary);
    margin-top: 0.4rem;
  }

  &__content {
    display: flex;
    flex-direction: column;
    gap: 1.2rem;
  }
}
</style>
