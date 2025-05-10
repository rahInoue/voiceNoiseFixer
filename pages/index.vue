<template>
  <client-only>
    <div class="container mx-auto p-4">
      <h1 class="text-2xl font-bold mb-4">まるっとボイス エディター</h1>
      <input type="file" accept="audio/*" @change="onFileChange" class="mb-4" />
    </div>
  </client-only>
</template>

<script setup>
</script>

<style scoped>
.container { max-width: 800px; }
label { display: block; margin-bottom: .5rem; }
button { margin-top: 1rem; }
</style>