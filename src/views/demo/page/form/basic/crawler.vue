<template>
  <div class="iframe-container">
    <iframe
      src="http://10.3.1.202:8080/#/tasks"
      frameborder="0"
      width="100%"
      height="100%"
    ></iframe>
  </div>
</template>

<script setup>
// 无需 JS 脚本，除非你想动态控制 iframe 的 src 等参数
</script>

<style scoped>
.iframe-container {
  width: 100%;
  height: 100vh; /* 你可以根据需要设置具体高度 */
  overflow: hidden;
}
</style>
