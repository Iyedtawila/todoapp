<template>
  <div class="app">
    <Todo />
  </div>
</template>

<script setup lang="ts">
import Todo from './components/Todo.vue'
</script>

<style>
.app {
  min-height: 100vh;
  background-color: #f5f5f5;
  padding: 20px;
}

body {
  margin: 0;
  font-family: Arial, sans-serif;
}
</style>
