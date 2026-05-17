<script setup lang="ts">
// placeholder — 第一版骨架，內容會在後續 commit 補
const status = 'init'
</script>

<template>
  <div class="wishlist">
    <p class="kicker">WISHLIST · {{ status }}</p>
    <p>UI 載入中…</p>
  </div>
</template>

<style scoped>
.wishlist {
  padding: 48px;
  color: var(--tx-2, #ccc);
  font-family: inherit;
}
.kicker {
  font-size: 11px;
  letter-spacing: 2px;
  color: var(--tx-3, #888);
  margin-bottom: 8px;
}
</style>
