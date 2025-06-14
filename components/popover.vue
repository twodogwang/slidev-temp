<template>
  <n-popover trigger="click">
    <template #trigger>
      <slot name="trigger"></slot>
    </template>
    <slot name="default"></slot>
  </n-popover>
</template>

<script setup lang="ts">
import { NPopover } from 'naive-ui';

</script>