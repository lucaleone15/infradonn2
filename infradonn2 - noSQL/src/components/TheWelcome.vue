<script setup lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

// DATA VARIABLE
const counter = ref(0)

// FUNCTIONS / METHODS => CONTROLLER

function initDatabase() {
  const db = new PouchDB('http://admin:admin@localhost:5984/test_infradonn2')
}

function mounted() {
  initDatabase()
}

const increment = () => {
  counter.value++
}
</script>

<template>
  <h1>Welcome</h1>
  <p>Counter : {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
