<script setup lang="ts">
    const { hash } = defineProps<{ hash: string }>();
</script>

<template>
    (<a href="https://github.com/TeamZincs/Kipphi-player/commit/{{ hash }}">{{ hash }}</a>)
</template>