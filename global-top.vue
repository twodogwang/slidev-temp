<script setup lang="ts">
import Progress from './components/Progress.vue'
import { useNav, useSlideContext } from '@slidev/client'
import { computed } from 'vue'

const { currentPage, total } = useNav()

const { $slidev } = useSlideContext()

const num = computed(() => {
  const p = (currentPage.value / total.value) * 100
  return Math.floor(p)
})
</script>

<template>
  <div v-if="![1, 2, $slidev.nav.total + 1].includes($slidev.nav.currentPage)">
    <div class="top absolute top-0 right-0 p-2">
      <!-- <img src="/share/giga.png" alt="logo" class="h-6" /> -->
    </div>
  </div>
</template>

<style>
.top {
  color: rgb(55, 95, 255);
  font-size: 32px;
}
.progress {
  top: -20px;
  left: -45px;
}
</style>
