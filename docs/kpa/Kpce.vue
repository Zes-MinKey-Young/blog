<script setup lang="ts">
    const { hash } = defineProps<{ hash: string }>();
</script>

<template>
    (<a href="https://github.com/TeamZincs/Kipphi-canvas-editor/commit/{{ hash }}">{{ hash }}</a>)
</template>