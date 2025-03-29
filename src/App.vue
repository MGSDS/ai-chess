<script setup>
import ChessBoard from './components/ChessBoard.vue'
</script>

<template>
  <main>
    <h1>Chess Game</h1>
    <ChessBoard />
  </main>
</template>

<style scoped>
main {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 2rem;
}

h1 {
  margin-bottom: 2rem;
  color: #333;
}
</style>
