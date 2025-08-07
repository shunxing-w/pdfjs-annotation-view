<template>
  <div class="pdf-page-container">
    <h3>PDF 查看器</h3>
    <PdfViewer
      :src="pdfSrc"
      :showToolbar="true"
      :showAnnotationTools="true"
      viewerMode="canvas"
      :initialPage="1"
      :initialScale="1.0"
      :annotationData="annotationData"
      @saveAnnotations="onSaveAnnotations"
    />
  </div>
</template>

<script setup lang="ts">
import { ref,onMounted } from 'vue';
import PdfViewer from '../components/PdfViewer.vue';

// 标注数据，可以从API获取
const annotationData = ref<Record<string, any>>([]);
// PDF文件路径，可以从路由参数或API获取
const pdfSrc ='http://localhost:5173/EUPL.pdf';
const onSaveAnnotations = (annotations: any) => {
  console.log('保存标注:', annotations);
  localStorage.setItem('annotations', JSON.stringify(annotations));
}

const onLoadAnnotations = () => {
  const annotations = localStorage.getItem('annotations');
  if (annotations) {
    annotationData.value = JSON.parse(annotations);
  }
}
onMounted(() => {
  onLoadAnnotations();
})
</script>

<style scoped>
.pdf-page-container {
  width: 100%;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

h1 {
  text-align: center;
  padding: 20px 0;
  margin: 0;
}
</style>