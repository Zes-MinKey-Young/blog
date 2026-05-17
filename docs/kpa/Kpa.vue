<script setup lang="ts">
    const { hash } = defineProps<{ hash: string }>();
</script>

<template>
    (<a href="https://github.com/TeamZincs/Kipphi-apparatus/commit/{{ hash }}">{{ hash }}</a>)
</template>