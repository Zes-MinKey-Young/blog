<script setup lang="ts">
    const { version } = defineProps<{
        version: string;
    }>();
</script>

<template>
    <ul style="border-radius: 5px; padding: 8px; padding-left: 32px; background-color: #6cf5;">
        <li>获得可执行程序：<a href="https://github.com/TeamZincs/kipphi-apparatus/releases/tag/app-{{ version }}">GitHub</a></li>
        <li>阅读 Wiki 文章：<a href="https://pgrfm.miraheze.org/wiki/Tutorial:奇谱发生器/从Re:PhiEdit迁移">Phigros 自制谱 Wiki</a></li>
        <li>加入官群：478824121</li>
    </ul>
</template>
