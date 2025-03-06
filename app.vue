<template>
  <div class="min-h-screen bg-gray-50">
    <TheHeader />
    <NuxtPage />
    <TheFooter />
    <div id="toast-container"></div>
  </div>
</template>

<script setup>
import Toast from 'vue-toastification'
import "vue-toastification/dist/index.css"

const { useToast } = Toast
const toast = useToast()

// Make toast available globally
provide('toast', toast)
</script>