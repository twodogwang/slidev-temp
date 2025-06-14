<template>
    <n-modal v-model:show="showModal" :mask-closable="false" preset="dialog" title="Dialog" style="width: 600px">
      <slot></slot>
    </n-modal>
</template>

<script setup lang="ts">
import { NModal } from 'naive-ui'
import { ref, watch } from 'vue';
const props = defineProps<{
  active: any
}>()
const showModal = ref(false)
watch(
  () => props.active,
  (val) => {
    showModal.value = val
  }
)
</script>

<style scoped>
.carousel-img {
  width: 100%;
  height: 440px;
  object-fit: cover;
}
</style>
