<script setup>
</script>

<template>
  <div class="app">

    <router-view></router-view>
  </div>
</template>

<style scoped>
.app {
  /* max-width: 1280px; */
  width: 100%;
  margin: 0 auto;
  /* padding: 2rem; */
  text-align: center;
}

.logo-container {
  display: flex;
  justify-content: center;
  gap: 2rem;
  margin-bottom: 2rem;
}

.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style>
