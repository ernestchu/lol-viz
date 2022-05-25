<script setup>
import { RouterView, RouterLink } from 'vue-router'
</script>

<template>
  <h1>
    <RouterLink :to="{ name: 'home' }">
      League of Legends Visualization
    </RouterLink>
  </h1>
  <RouterView />
</template>

<style>
h1 {
  margin-top: 0;
}
</style>
