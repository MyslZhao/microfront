<script setup>
import { onMounted, ref } from 'vue'

const message = ref('')

onMounted(async () => {
  try {
    const response = await fetch('/api/hello')
    const text = await response.text()
    message.value = text
  } catch (error) {
    console.error('API call failed:', error)
    message.value = 'Failed to load message.'
  }
})
</script>

<template>
  <!-- NOTE: Test starts -->
  <main>
    <h1>{{ message }}</h1>
    <input id="test_msg" />
    <button type="submit">提交</button>
  </main>
  <!-- NOTE: Test ends -->
</template>
