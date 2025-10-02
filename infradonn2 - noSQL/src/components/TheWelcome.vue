<script setup lang="ts">
import { ref } from 'vue'
// DATA VARIABLE
const counter = ref(0)

// FUNCTIONS / METHODS => CONTROLLER
const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>Welcome</h1>
  <p>Counter : {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
