<template>
  <div class="home">
    <h1>欢迎来到PDFJS预览项目</h1>
    <p>这是一个使用Vite+Vue3+pdfjs-dist 构建的项目PDFJS预览项目</p>
    <p>支持pdf文件的预览、标注、保存等功能</p>
    <p>标注功能支持：文字、矩形、圆形、箭头、直线、多边形等</p>
    <p>标注数据支持本地存储</p>
    <router-link to="/pdf">前往pdf预览页面</router-link>
  </div>
</template>

<script setup>
// 首页组件逻辑
</script>

<style scoped>
.home {
  padding: 2rem;
  text-align: center;
}

h1 {
  color: #42b983;
}

router-link {
  display: inline-block;
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background-color: #42b983;
  color: white;
  text-decoration: none;
  border-radius: 4px;
}
</style>