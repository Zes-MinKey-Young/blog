<script setup lang="ts">
    const { hash } = defineProps<{ hash: string }>();
</script>

<template>
    (<a href="https://github.com/TeamZincs/Kipphi/commit/{{ hash }}">{{ hash }}</a>)
</template>