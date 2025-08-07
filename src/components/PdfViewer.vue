<template>
  <div class="pdf-viewer-container">
    <!-- 主工具栏 -->
    <div class="pdf-toolbar" v-if="showToolbar">
      <!-- 移动端工具栏折叠按钮 -->
      <!-- <button class="toolbar-toggle mobile-only" @click="toggleToolbarExpanded">
        <Icon :icon="toolbarExpanded ? 'ri-arrow-up-line' : 'ri-arrow-down-line'" />
      </button> -->
      
      <div class="toolbar-content">
        <div class="toolbar-section toolbar-navigation">
          <button @click="previousPage" :disabled="currentPage <= 1" class="toolbar-btn">
           上一页
          </button>
          <span class="page-info">
            <input
              v-model="pageInput"
              @keyup.enter="goToPage"
              type="number"
              :min="1"
              :max="totalPages"
              class="page-input-inline"
            />
            <span class="page-total">/ {{ totalPages }}</span>
          </span>
          <button @click="nextPage" :disabled="currentPage >= totalPages" class="toolbar-btn">
           下一页
          </button>
        </div>
        
        <div class="toolbar-section toolbar-zoom">
          <button @click="zoomOut" class="toolbar-btn">
           缩小
          </button>
          <!-- <span class="zoom-info">{{ Math.round(scale * 100) }}%</span> -->
           <div class="zoom-select-box">
            <select 
              v-model="selectedZoomOption" 
              @change="handleZoomChange" 
              class="zoom-select"
              title="缩放">
              <option value="auto">自动缩放</option>
              <option value="page-actual">实际大小</option>
              <option value="page-fit">适合页面</option>
              <option value="page-width">适合页宽</option>
              <option value="0.5">50%</option>
              <option value="0.75">75%</option>
              <option value="1">100%</option>
              <option value="1.25">125%</option>
              <option value="1.5">150%</option>
              <option value="2">200%</option>
              <option value="3">300%</option>
              <option value="4">400%</option>
              <option value="custom" disabled>{{ Math.round(scale * 100) }}%</option>
            </select>
           </div>
          <button @click="zoomIn" class="toolbar-btn">
           放大
          </button>
        </div>
        
        <div class="toolbar-section toolbar-actions">
          <button @click="rotateLeft" class="toolbar-btn">
            <span class="btn-text">左转</span>
          </button>
          <button @click="rotateRight" class="toolbar-btn">
            <span class="btn-text">右转</span>
          </button>
          <button @click="toggleFullscreen" class="toolbar-btn">
            <span class="btn-text">{{ isFullscreen ? '退出' : '全屏' }}</span>
          </button>
        </div>
      </div>
    </div>

    <!-- 标注工具栏 -->
    <div class="annotation-toolbar" v-if="showAnnotationTools">
      <div class="annotation-toolbar-scroll">
        <div class="annotation-tools">
          <!-- 标注工具按钮 -->
          <div class="tool-wrapper" v-for="tool in annotationTools" :key="tool.type">
            <button
              @click="toggleTool(tool.type, $event)"
              :class="['tool-btn', { 'active': currentTool === tool.type }]"
              :data-tool="tool.type"
              :title="tool.title"
            >
              <span class="tool-name">{{ tool.name }}</span>
            </button>
          </div>

              <!-- 悬浮工具设置面板 -->
          <div class="floating-tool-settings" v-if="currentTool&&currentTool!='delete'" :class="`tool-${currentTool}`" :style="{
              top: `${toolSettingsTop}px`,
              left: `${toolSettingsLeft}px`
            }">
            <!-- 高亮工具设置 -->
            <div v-if="currentTool === 'highlight'||currentTool === 'underline'" class="tool-settings">
              <div class="setting-group">
                <label>颜色:</label>
                <input
                  type="color"
                  v-model="currentColor"
                  class="color-picker"
                  title="标注颜色"
                />
              </div>
            </div>
            
            <!-- 文本框工具设置 -->
            <div v-if="currentTool === 'text'" class="tool-settings">
              <div class="setting-group">
                <label>颜色:</label>
                <input
                  type="color"
                  v-model="currentTextColor"
                  class="color-picker"
                  title="文字颜色"
                />
              </div>
              <div class="setting-group">
                <label>大小:</label>
                <select v-model="currentFontSize" class="font-size-select">
                  <option value="10">10px</option>
                  <option value="12">12px</option>
                  <option value="14">14px</option>
                  <option value="16">16px</option>
                  <option value="18">18px</option>
                  <option value="20">20px</option>
                  <option value="24">24px</option>
                </select>
              </div>
            </div>
            
            <!-- 画笔工具设置 -->
            <div v-if="currentTool === 'draw'" class="tool-settings">
              <div class="setting-group">
                <label>颜色:</label>
                <input
                  type="color"
                  v-model="currentDrawColor"
                  class="color-picker"
                  title="画笔颜色"
                />
              </div>
              <div class="setting-group">
                <label>粗细:</label>
                <select v-model="currentLineWidth" class="line-width-select">
                  <option value="1">1px</option>
                  <option value="2">2px</option>
                  <option value="3">3px</option>
                  <option value="4">4px</option>
                  <option value="5">5px</option>
                  <option value="8">8px</option>
                </select>
              </div>
            </div>
          </div>
        </div>      
        
        <div class="annotation-actions">
          <button @click="onShowAnnotations" class="action-btn load-btn" title="显示标注">
            <span v-if="isShowAnnotation" class="action-text">显示</span>
            <span v-else class="action-text">隐藏</span>
          </button>
          <!-- <button @click="clearAllAnnotations" class="action-btn clear-btn" title="隐藏标注">
          
          </button> -->
          <button @click="submitAnnotations" class="action-btn submit-btn" title="保存标注">
            <span class="action-text">保存</span>
          </button>
          <button @click="toggleAnnotationList" class="action-btn submit-btn" title="标注列表">
            标注列表
          </button>
        </div>
      </div>
    </div>
    <!-- 主内容区域 -->
    <div class="pdf-main-content">
      <!-- PDF内容区域 -->
      <div class="pdf-content" ref="pdfContainer" :class="{ 'fullscreen': isFullscreen }">
        <!-- 使用iframe方式 -->
        <iframe 
          v-if="viewerMode === 'iframe'"
          ref="pdfIframe"
          :src="iframeSrc"
          class="pdf-iframe"
          @load="onIframeLoad"
        ></iframe>        
        <!-- 使用canvas渲染方式 -->
        <div v-else-if="viewerMode === 'canvas'" class="pdf-canvas-container">
          <div v-if="loading" class="loading-container">
            <div class="loading-spinner"></div>
            <p>正在加载PDF...</p>
          </div>
          <div class="all-pages-container" ref="allPagesContainer"
          @mousedown="handleDragStart"
          @mouseup="handleDragEnd"
          @mouseleave="handleDragEnd"
          :style="{ 
            cursor: isDragging ? 'grabbing' : 'grab',
            transition: isDragging ? 'none' : 'transform 0.2s ease'
          }"
          >
            <div 
              v-for="pageNum in totalPages" 
              :key="pageNum"
              class="pdf-page"
              :data-page-num="pageNum"
              :style="getPageStyle(pageNum)"
            >
              <canvas 
                :ref="(el) => setPageCanvas(pageNum, el as HTMLCanvasElement)"
                class="pdf-canvas"
              ></canvas>
              <!-- 标注层 -->
              <div class="annotation-layer"></div>
              <!-- 交互层 -->
              <div 
                class="interaction-layer"
                @mousedown="(e) => handleMouseDown(e, pageNum)"
                @mousemove="(e) => handleMouseMove(e, pageNum)"
                @mouseup="(e) => handleMouseUp(e, pageNum)"
                @click="(e) => handleClick(e, pageNum)"
                @touchstart="(e) => handleTouchStart(e, pageNum)"
                @touchmove="(e) => handleTouchMove(e, pageNum)"
                @touchend="(e) => handleTouchEnd(e, pageNum)"
              ></div>
            </div>
          </div>
        </div>
        
        <!-- 错误提示 -->
        <div v-if="error" class="error-container">
          <p class="error-message">{{ error }}</p>
          <button @click="retry" class="retry-btn">重试</button>
        </div>
      </div>

      <!-- 标注列表抽屉 -->
      <div class="annotation-drawer-overlay"
           v-if="showAnnotationList && annotations.length > 0"
           @click="toggleAnnotationList"
           :class="{ 'show': showAnnotationList }">
      </div>
      
      <div class="annotation-drawer"
           v-if="annotations.length > 0"
           :class="{ 'show': showAnnotationList }"
           @click.stop>
        
        <div class="drawer-header">
          <h3>标注列表 ({{ annotations.length }})</h3>
          <button @click="toggleAnnotationList" class="close-btn">×</button>
        </div>
        
        <div class="drawer-content">
          <div class="annotation-list">
            <div
              v-for="annotation in annotations"
              :key="annotation.id"
              class="annotation-item"
              :class="{ 'selected': selectedAnnotationId === annotation.id }"
              @click="selectAnnotation(annotation)"
            >
              <div class="annotation-header">
                <div>
                  <span class="annotation-type">{{ getAnnotationTypeName(annotation.type) }}</span>
                </div>                
                <button  @click.stop="deleteAnnotation(annotation.id)" class="delete-annotation-btn">
                  删除
                </button>
              </div>
              <div class="annotation-content" v-if="annotation.text">{{ annotation.text }}</div>
              <div class="annotation-time">{{ formatTime(annotation.annotationTime||annotation.timestamp) }} <span class="annotation-page">第{{ annotation.pageNum }}页</span> </div>
            </div>
          </div>
        </div>
        <div class="drawer-footer">
          <button @click="submitAnnotations"  class="submit-btn">保存标注</button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted, watch, nextTick, shallowRef } from 'vue'
import * as pdfjsLib from 'pdfjs-dist'
import PdfWorker from "pdfjs-dist/build/pdf.worker.min.mjs?url"

// 配置PDF.js worker
pdfjsLib.GlobalWorkerOptions.workerSrc = PdfWorker

// 标注接口定义
interface Annotation {
  id: string | number
  // type: 'highlight' | 'underline' | 'text' | 'draw' | 'delete'
   type: 'highlight' | 'underline' | 'text' | 'draw' | 'delete' | 'rectangle' | 'circle' | 'triangle' | 'arrow'
  pageNum: number
  // 标准化坐标 (0-1之间的比例，相对于PDF页面)
  normalizedX?: number
  normalizedY?: number
  normalizedWidth?: number
  normalizedHeight?: number
  // 圆形专用属性
  centerX?: number
  centerY?: number
  radius?: number
  // 标准化路径坐标 (用于手绘)
  normalizedPath?: Array<{ x: number; y: number }>
  text?: string
  color?: string
  textColor?: string
  fontSize?: string
  lineWidth?: string
  timestamp: string
  isEditing?: boolean
  doctorId?: number
  doctorName?: string
  annotationTime?: string
}

interface Props {
  src: string // PDF文件路径或URL
  showToolbar?: boolean // 是否显示工具栏
  showAnnotationTools?: boolean // 是否显示标注工具
  viewerMode?: 'iframe' | 'canvas' // 查看器模式
  initialPage?: number // 初始页码
  initialScale?: number // 初始缩放比例
  annotationData?: Record<string, any> 
}

const props = withDefaults(defineProps<Props>(), {
  showToolbar: true,
  showAnnotationTools: true,
  viewerMode: 'canvas',
  initialPage: 1,
  initialScale: 1.0,
  annotationData: () => ({})
})

// 基础响应式数据
const pdfContainer = ref<HTMLElement>()
const pdfIframe = ref<HTMLIFrameElement>()
const allPagesContainer = ref<HTMLElement>()
const loading = ref(false)
const error = ref('')
const currentPage = ref(props.initialPage)
const totalPages = ref(0)
const scale = ref(props.initialScale)
const pageInput = ref(props.initialPage)
const isFullscreen = ref(false)
const pdfDocument = shallowRef<any>(null)
const pages = shallowRef<{pageNum: number; canvas: HTMLCanvasElement | null}[]>([])

// PDF 工具栏相关属性
const rotation = ref(0)
const fitMode = ref('page') // 'page', 'width', 'auto'
const highQualityMode = ref(false)

// 标注相关状态
const annotations = ref<Annotation[]>([])
const currentTool = ref<string>('')
const currentColor = ref('#ffff00')
const currentTextColor = ref('#000000')
const currentFontSize = ref('12')
const currentDrawColor = ref('#ff0000')
const currentLineWidth = ref('2')
const selectedAnnotation = ref<Annotation | null>(null)
const showAnnotationList = ref(false)
const isShowAnnotation = ref(true)
const selectedAnnotationId = ref<string | number | null>(null)

// 移动端工具栏状态
const toolbarExpanded = ref(false)

// 交互状态
const isDrawing = ref(false)
const drawingPath = ref<Array<{ x: number; y: number }>>([])
const startX = ref(0)
const startY = ref(0)
const currentFileName = ref('')

// 文本框拖动状态
const isDraggingTextbox = ref(false)
const draggedTextbox = ref<Annotation | null>(null)
const dragStartX = ref(0)
const dragStartY = ref(0)
const dragOffsetX = ref(0)
const dragOffsetY = ref(0)


// 添加拖动相关状态
const isDragging = ref(false)
const scrollOffsetX = ref(0)
const scrollOffsetY = ref(0)

// 触摸拖动相关状态
const touchStartX = ref(0)
const touchStartY = ref(0)


// 标注工具配置
const annotationTools = ref([
  { type: 'highlight', name: '高亮', title: '高亮文本' },
  { type: 'underline', name: '下划线', title: '下划线文本' },
  { type: 'text', name: '文本框', title: '添加文本框' },
  { type: 'draw', name: '画笔', title: '画笔标注' },
  { type: 'triangle', name: '三角形', title: '三角形标注' },
  { type: 'circle', name: '圆形', title: '圆形标注' },
  { type: 'arrow', name: '箭头', title: '箭头标注' },
  { type: 'rectangle', name: '矩形', title: '矩形标注' },
  { type: 'delete', name: '删除', title: '删除标注' }
])

const selectedZoomOption = ref('auto')  // 默认选择自动缩放

// 计算属性
const iframeSrc = computed(() => {
  if (props.viewerMode === 'iframe' && props.src) {
    return `/pdfWebViewer/web/viewer.html?file=${encodeURIComponent(props.src)}`
  }
  return ''
})

// 设置页面canvas引用
const setPageCanvas = (pageNum: number, el: HTMLCanvasElement | null) => {
  // console.log('setPageCanvas', pageNum, el)
  if (el) {
    pages.value[pageNum - 1] = { pageNum, canvas: el }
  }
}

// 获取页面样式
const getPageStyle = (pageNum: number) => {
  const style: any = {
    position: 'relative',
    margin: '0 auto',
    background: 'white',
    boxShadow: '0 2px 8px rgba(0,0,0,0.15)'
  }
  
  if (rotation.value !== 0) {
    style.transform = `rotate(${rotation.value}deg)`
    style.transformOrigin = 'center center'
  }
  
  return style
}

const toolSettingsTop = ref(0)
const toolSettingsLeft = ref(0)

const toggleTool = (tool: string, event: MouseEvent) => {
  // 如果点击的是当前已选中的工具，则取消选中
  if (currentTool.value === tool) {
    currentTool.value = '';
  } else {
    currentTool.value = tool;
    // 获取点击按钮的位置
    const button = event.target as HTMLElement
    const buttonRect = button.getBoundingClientRect()

    // 获取 annotation-toolbar-scroll 的位置
    const toolbarRect = document.querySelector('.annotation-toolbar-scroll')?.getBoundingClientRect()
    if (toolbarRect) {
      // 计算相对于 annotation-toolbar-scroll 的位置
      toolSettingsTop.value = buttonRect.bottom - toolbarRect.top+20
      toolSettingsLeft.value = buttonRect.left - toolbarRect.left+25
    }
  }
}

// PDF 加载和渲染方法
const loadPDF = async () => {
  if (!props.src) {
    error.value = 'PDF文件路径不能为空'
    return
  }

  loading.value = true
  error.value = ''

  try {
    const loadingTask = pdfjsLib.getDocument({
      url: props.src,
      cMapUrl: '/pdfWebViewer/web/cmaps/',
      cMapPacked: true
    })
    pdfDocument.value = await loadingTask.promise
    totalPages.value = pdfDocument.value.numPages
    console.log( props.src,'PDF加载成功')
    // 从URL中提取文件名
    currentFileName.value = 'document.pdf'
    
    if (props.viewerMode === 'canvas') {
        nextTick(async () => { 
          calculateFitZoom()
          await renderAllPages()
          loadAnnotations()
        })
    
    }
  } catch (err) {
    console.error('PDF加载失败:', err)
    error.value = `PDF加载失败: ${err instanceof Error ? err.message : '未知错误'}`
  } finally {
    loading.value = false
  }
}

const renderPage = async (pageNum: number) => {
  console.log('renderPage', pageNum,pages.value[pageNum - 1],pages.value,pages.value[0])
  if (!pdfDocument.value || !pages.value[pageNum - 1]?.canvas) return
console.log('renderPage-------')
  try {
    const page = await pdfDocument.value.getPage(pageNum)
    const baseScale = scale.value
    const viewport = page.getViewport({ scale: baseScale })
    // const viewport = page.getViewport({ scale: baseScale, rotation: rotation.value })
    
    const canvas = pages.value[pageNum - 1].canvas!
    const context = canvas.getContext('2d')!
    
    // 获取设备像素比，提高清晰度
    const pixelRatio = window.devicePixelRatio || 1
    let ratio = pixelRatio
    if (highQualityMode.value) {
      ratio = Math.max(ratio, 2.0)
    }
    
    // 设置Canvas的实际尺寸
    canvas.width = viewport.width * ratio
    canvas.height = viewport.height * ratio
    
    // 设置Canvas的显示尺寸
    canvas.style.width = viewport.width + 'px'
    canvas.style.height = viewport.height + 'px'
    
    // 缩放绘图上下文以匹配设备像素比
    context.scale(ratio, ratio)
    
    // 设置Canvas渲染优化
    context.imageSmoothingEnabled = true
    context.imageSmoothingQuality = highQualityMode.value ? 'high' : 'medium'

    const renderContext = {
      canvasContext: context,
      viewport: viewport
    }

    await page.render(renderContext).promise
  } catch (err) {
    console.error(`页面${pageNum}渲染失败:`, err)
    error.value = `页面${pageNum}渲染失败: ${err instanceof Error ? err.message : '未知错误'}`
  }
}

const renderAllPages = async () => {
  if (props.viewerMode === 'canvas' && pdfDocument.value) {

     // 重置拖动偏移
     scrollOffsetX.value = 0
    scrollOffsetY.value = 0
    if (allPagesContainer.value) {
      allPagesContainer.value.style.transform = 'translate(0, 0)'
    }

    for (let i = 1; i <= totalPages.value; i++) {
      console.log(`开始渲染页面${i}`)
      await renderPage(i)
    }
    // 重新渲染标注
    renderAllAnnotations()
  }
}

// 页面导航方法
const previousPage = () => {
  if (currentPage.value > 1) {
    currentPage.value--
    pageInput.value = currentPage.value
    scrollToPage(currentPage.value)
  }
}

const nextPage = () => {
  if (currentPage.value < totalPages.value) {
    currentPage.value++
    pageInput.value = currentPage.value
    scrollToPage(currentPage.value)
  }
}

const goToPage = () => {
  const page = parseInt(pageInput.value.toString())
  if (page >= 1 && page <= totalPages.value) {
    currentPage.value = page
    scrollToPage(page)
  } else {
    pageInput.value = currentPage.value
  }
}

const scrollToPage = (pageNum: number) => {
  const canvas = pages.value[pageNum - 1]?.canvas
  if (canvas) {
    canvas.scrollIntoView({ behavior: 'smooth', block: 'start' })
  }
}

// 添加缩放选择框变更处理方法
const handleZoomChange = () => {
  switch(selectedZoomOption.value) {
    case 'auto':
      calculateFitZoom()  // 自动缩放
      break
    case 'page-actual':
      scale.value = 1.0  // 实际大小(100%)
      renderAllPages()
      break
    case 'page-fit':
      fitToPage()  // 适合页面
      break
    case 'page-width':
      fitToWidth()  // 适合页宽
      break
    default:
      // 百分比缩放
      const zoomValue = parseFloat(selectedZoomOption.value)
      if (!isNaN(zoomValue)) {
        scale.value = zoomValue
        renderAllPages()
      }
  }
}

// 根据当前缩放值更新选择框
const updateZoomSelect = () => {
  // 检查是否匹配预设缩放值
  const scaleOptions = [0.5, 0.75, 1, 1.25, 1.5, 2, 3, 4]
  const closestValue = scaleOptions.reduce((prev, curr) => 
    Math.abs(curr - scale.value) < Math.abs(prev - scale.value) ? curr : prev
  )
  console.log('closestValue',closestValue)
  console.log('scale.value',scale.value,Math.abs(closestValue - scale.value))
  
  // 如果接近预设值，则选择预设值，否则保持当前选择
  selectedZoomOption.value = 'custom'
}

// 缩放控制方法
const zoomIn = () => {
  scale.value = Math.min(scale.value * 1.2, 5.0)
  renderAllPages()
  updateZoomSelect()

}

const zoomOut = () => {
  scale.value = Math.max(scale.value / 1.2, 0.1)
  renderAllPages()
  updateZoomSelect()

}

const resetZoom = () => {
  scale.value = 1.0
  renderAllPages()
}

const fitToWidth = () => {
  fitMode.value = 'width'
  calculateFitZoom()
  renderAllPages()
}

const fitToPage = () => {
  fitMode.value = 'page'
  calculateFitZoom()
  renderAllPages()
}

const calculateFitZoom = async  () => {
  const container = pdfContainer.value
  if (!container || !pdfDocument.value) return

  const containerWidth = container.clientWidth
  const containerHeight = container.clientHeight
  try {
    // 获取第一页 PDF 的信息
    const firstPage = await pdfDocument.value.getPage(1)
    const viewport = firstPage.getViewport({ scale: 1.0 })
    const pdfPageWidth = viewport.width+40

    // 根据容器宽度和 PDF 页面宽度计算缩放比例
    scale.value = containerWidth / pdfPageWidth
  } catch (err) {
    console.error('获取 PDF 页面信息失败:', err)
    error.value = `获取 PDF 页面信息失败: ${err instanceof Error ? err.message : '未知错误'}`
  }
}

// 旋转控制方法
const rotateLeft = () => {
  rotation.value = (rotation.value - 90) % 360
  renderAllPages()
}

const rotateRight = () => {
  rotation.value = (rotation.value + 90) % 360
  renderAllPages()
}

// 全屏控制
const toggleFullscreen = () => {
  isFullscreen.value = !isFullscreen.value
  
  if (isFullscreen.value) {
    if (pdfContainer.value?.requestFullscreen) {
      pdfContainer.value.requestFullscreen()
    }
  } else {
    if (document.exitFullscreen) {
      document.exitFullscreen()
    }
  }
}

// 高质量模式切换
const toggleHighQuality = () => {
  highQualityMode.value = !highQualityMode.value
  renderAllPages()
}

// 移动端工具栏展开切换
const toggleToolbarExpanded = () => {
  toolbarExpanded.value = !toolbarExpanded.value
}

// iframe 加载处理
const onIframeLoad = () => {
  loading.value = false
}

// 重试方法
const retry = () => {
  error.value = ''
  loadPDF()
}

// 标注工具方法
const setCurrentTool = (tool: string) => {
  currentTool.value = tool
}

// 获取正确的Canvas坐标的辅助函数
const getCanvasCoordinates = (e: MouseEvent, pageNum: number) => {
  const canvas = pages.value[pageNum - 1]?.canvas
  if (!canvas) return { x: 0, y: 0 }
  
  const canvasRect = canvas.getBoundingClientRect()
  
  // 计算相对于canvas的坐标
  const canvasX = e.clientX - canvasRect.left
  const canvasY = e.clientY - canvasRect.top
  
  // 获取canvas的显示尺寸和实际尺寸
  const displayWidth = parseFloat(canvas.style.width)
  const displayHeight = parseFloat(canvas.style.height)
  const actualWidth = canvas.width
  const actualHeight = canvas.height
  
  // 计算缩放比例（考虑设备像素比）
  const scaleX = actualWidth / displayWidth
  const scaleY = actualHeight / displayHeight
  
  // 转换为canvas内部坐标
  return {
    x: canvasX * scaleX,
    y: canvasY * scaleY
  }
}

// 坐标转换辅助函数
const convertToNormalizedCoordinates = (pageNum: number, canvasX: number, canvasY: number) => {
  const canvas = pages.value[pageNum - 1]?.canvas
  if (!canvas || !pdfDocument.value) return { x: 0, y: 0 }
  
  // 获取canvas的显示尺寸
  const displayWidth = parseFloat(canvas.style.width)
  const displayHeight = parseFloat(canvas.style.height)
  
  // 获取设备像素比
  const pixelRatio = window.devicePixelRatio || 1
  let ratio = pixelRatio
  if (highQualityMode.value) {
    ratio = Math.max(ratio, 2.0)
  }
  
  // 将canvas内部坐标转换为显示坐标
  const displayX = canvasX / ratio
  const displayY = canvasY / ratio
  
  // 将显示坐标转换为标准化坐标 (0-1)
  const normalizedX = displayX / displayWidth
  const normalizedY = displayY / displayHeight
  
  return {
    x: Math.max(0, Math.min(1, normalizedX)),
    y: Math.max(0, Math.min(1, normalizedY))
  }
}

const convertFromNormalizedCoordinates = (pageNum: number, normalizedX: number, normalizedY: number) => {
  const canvas = pages.value[pageNum - 1]?.canvas
  if (!canvas || !pdfDocument.value) return { x: 0, y: 0 }
  
  // 获取当前显示尺寸
  const displayWidth = parseFloat(canvas.style.width)
  const displayHeight = parseFloat(canvas.style.height)
  
  // 将标准化坐标转换为当前显示坐标
  const displayX = normalizedX * displayWidth
  const displayY = normalizedY * displayHeight
  
  return { x: displayX, y: displayY }
}

// 获取指定位置的文本框标注
const getTextboxAtPosition = (pageNum: number, canvasX: number, canvasY: number) => {
  const normalized = convertToNormalizedCoordinates(pageNum, canvasX, canvasY)
  
  return annotations.value.find(annotation => {
    if (annotation.pageNum !== pageNum || annotation.type !== 'text') return false
    
    const x1 = annotation.normalizedX!
    const y1 = annotation.normalizedY!
    const x2 = x1 + annotation.normalizedWidth!
    const y2 = y1 + annotation.normalizedHeight!
    
    return normalized.x >= x1 && normalized.x <= x2 &&
           normalized.y >= y1 && normalized.y <= y2
  })
}

const convertSizeToNormalized = (pageNum: number, width: number, height: number) => {
  const canvas = pages.value[pageNum - 1]?.canvas
  if (!canvas) return { width: 0, height: 0 }
  
  // 获取canvas的显示尺寸
  const displayWidth = parseFloat(canvas.style.width)
  const displayHeight = parseFloat(canvas.style.height)
  
  // 获取设备像素比
  const pixelRatio = window.devicePixelRatio || 1
  let ratio = pixelRatio
  if (highQualityMode.value) {
    ratio = Math.max(ratio, 2.0)
  }
  
  // 将canvas内部尺寸转换为显示尺寸，再转换为标准化尺寸
  const displaySizeWidth = width / ratio
  const displaySizeHeight = height / ratio
  
  const normalizedWidth = displaySizeWidth / displayWidth
  const normalizedHeight = displaySizeHeight / displayHeight
  
  return {
    width: Math.max(0, Math.min(1, normalizedWidth)),
    height: Math.max(0, Math.min(1, normalizedHeight))
  }
}

const convertSizeFromNormalized = (pageNum: number, normalizedWidth: number, normalizedHeight: number) => {
  const canvas = pages.value[pageNum - 1]?.canvas
  if (!canvas) return { width: 0, height: 0 }
  
  const displayWidth = parseFloat(canvas.style.width)
  const displayHeight = parseFloat(canvas.style.height)
  
  return {
    width: normalizedWidth * displayWidth,
    height: normalizedHeight * displayHeight
  }
}

// 鼠标事件处理
const handleMouseDown = (e: MouseEvent, pageNum: number) => {
  if (currentTool.value === 'delete') return
  
  const coords = getCanvasCoordinates(e, pageNum)
  startX.value = coords.x
  startY.value = coords.y
  
  // 检查是否有正在编辑的文本框
  const editingTextbox = document.querySelector('.annotation-text[contenteditable="true"]:focus') as HTMLElement;
  
  if (editingTextbox) {
    // 如果点击的不是当前编辑的文本框，让其失焦
    const clickedElement = e.target as HTMLElement
    if (!editingTextbox.contains(clickedElement)) {
      editingTextbox.blur()
      return // 不允许创建新的标注
    }
  }
  
  // 检查是否点击了文本框（用于拖动）
  const clickedTextbox = getTextboxAtPosition(pageNum, coords.x, coords.y)
  if (clickedTextbox && currentTool.value === 'text') {
    // 开始拖动文本框
    isDraggingTextbox.value = true
    draggedTextbox.value = clickedTextbox
    dragStartX.value = coords.x
    dragStartY.value = coords.y
    
    // 计算鼠标相对于文本框的偏移
    const textboxPos = convertFromNormalizedCoordinates(pageNum, clickedTextbox.normalizedX!, clickedTextbox.normalizedY!)
    dragOffsetX.value = coords.x - textboxPos.x * (window.devicePixelRatio || 1)
    dragOffsetY.value = coords.y - textboxPos.y * (window.devicePixelRatio || 1)
    return
  }
  
  isDrawing.value = true
  
  if (currentTool.value === 'draw') {
    drawingPath.value = [{ x: startX.value, y: startY.value }]
  }
}

const handleMouseMove = (e: MouseEvent, pageNum: number) => {
  const coords = getCanvasCoordinates(e, pageNum)
  
  // 处理文本框拖动
  if (isDraggingTextbox.value && draggedTextbox.value) {
    const normalized = convertToNormalizedCoordinates(pageNum, coords.x - dragOffsetX.value, coords.y - dragOffsetY.value)
    
    // 更新文本框位置
    draggedTextbox.value.normalizedX = Math.max(0, Math.min(1 - draggedTextbox.value.normalizedWidth!, normalized.x))
    draggedTextbox.value.normalizedY = Math.max(0, Math.min(1 - draggedTextbox.value.normalizedHeight!, normalized.y))
    
    // 重新渲染该文本框
    const textboxElement = document.querySelector(`[data-annotation-id="${draggedTextbox.value.id}"]`) as HTMLElement
    if (textboxElement) {
      const position = convertFromNormalizedCoordinates(pageNum, draggedTextbox.value.normalizedX!, draggedTextbox.value.normalizedY!)
      textboxElement.style.left = position.x + 'px'
      textboxElement.style.top = position.y + 'px'
    }
    return
  }
  
  if (!isDrawing.value) return

  if (currentTool.value === 'draw') {
    drawingPath.value.push({ x: coords.x, y: coords.y })
    updateDrawingPreview(pageNum)
  }else if (currentTool.value === 'highlight' || currentTool.value === 'underline') {
    updateHighlightPreview(pageNum, startX.value, startY.value, coords.x, coords.y)
  } 
  else if (['rectangle', 'circle', 'triangle', 'arrow'].includes(currentTool.value)) {
    updateHighlightPreview(pageNum, startX.value, startY.value, coords.x, coords.y)
  }
}


const handleClick = (e: MouseEvent, pageNum: number) => {
  if (currentTool.value === 'delete') {
    const coords = getCanvasCoordinates(e, pageNum)
    console.log(coords)
    deleteAnnotationAt(pageNum, coords.x, coords.y)
  } else {
    // 点击空白区域取消选中
    clearAnnotationSelection()
  }
}

const handleMouseUp = (e: MouseEvent, pageNum: number) => {
  // 处理文本框拖动结束
  if (isDraggingTextbox.value) {
    isDraggingTextbox.value = false
    draggedTextbox.value = null
    return
  }
   // 移除高亮预览
   const pageDiv = document.querySelector(`[data-page-num="${pageNum}"]`)
  if (pageDiv) {
    const preview = pageDiv.querySelector('.highlight-preview')
    if (preview) preview.remove()
  }

  if (!isDrawing.value) return
  
  const coords = getCanvasCoordinates(e, pageNum)

  if (currentTool.value === 'highlight' || currentTool.value === 'underline') {
    createTextAnnotation(pageNum, startX.value, startY.value, coords.x, coords.y)
  } else if (currentTool.value === 'text') {
    createTextboxAnnotation(pageNum, startX.value, startY.value)
  } else if (currentTool.value === 'draw') {
    createDrawingAnnotation(pageNum)
  } else if (['rectangle', 'circle', 'triangle', 'arrow'].includes(currentTool.value)) {
    createShapeAnnotation(pageNum, currentTool.value, startX.value, startY.value, coords.x, coords.y)
  }

  isDrawing.value = false
  drawingPath.value = []
}

// 触摸事件处理函数
const handleTouchStart = (e: TouchEvent, pageNum: number) => {
  // 如果正在使用标注工具，不启用拖动
  // if (currentTool.value) return
  
  // 阻止默认滚动行为
  e.preventDefault()
  const touch = e.touches[0]

  if (currentTool.value) {
    const mouseEvent = new MouseEvent('mousedown', {
      clientX: touch.clientX,
      clientY: touch.clientY
    })
    handleMouseDown(mouseEvent, pageNum)
    return
  }

  touchStartX.value = touch.clientX
  touchStartY.value = touch.clientY
  isDragging.value = true
  const mouseEvent = new MouseEvent('mousedown', {
    clientX: touch.clientX,
    clientY: touch.clientY
  })
  handleMouseDown(mouseEvent, pageNum)
}

const handleTouchMove = (e: TouchEvent, pageNum: number) => {
  // 如果正在使用标注工具，不启用拖动
  // if (currentTool.value) return
  
  // 阻止默认滚动行为
  e.preventDefault()
  const touch = e.touches[0]
  if (currentTool.value) {
    const mouseEvent = new MouseEvent('mousemove', {
      clientX: touch.clientX,
      clientY: touch.clientY
    })
    handleMouseMove(mouseEvent, pageNum)
    return
  }
  const deltaX = touch.clientX - touchStartX.value
  const deltaY = touch.clientY - touchStartY.value

  scrollOffsetX.value += deltaX
  scrollOffsetY.value += deltaY

  touchStartX.value = touch.clientX
  touchStartY.value = touch.clientY

  // 应用滚动偏移
  if (allPagesContainer.value) {
    allPagesContainer.value.style.transform = `translate(${scrollOffsetX.value}px, ${scrollOffsetY.value}px)`
  }

  const mouseEvent = new MouseEvent('mousemove', {
    clientX: touch.clientX,
    clientY: touch.clientY
  })
  handleMouseMove(mouseEvent, pageNum)
}

const handleTouchEnd = (e: TouchEvent, pageNum: number) => {
  // 如果正在使用标注工具，不启用拖动
  // if (currentTool.value) return
  const touch = e.changedTouches[0]

  if (currentTool.value) {
    const mouseEvent = new MouseEvent('mouseup', {
      clientX: touch.clientX,
      clientY: touch.clientY
    })
    handleMouseUp(mouseEvent, pageNum)
      // 添加删除工具的触摸支持
      if (currentTool.value === 'delete') {
      const canvas = pages.value[pageNum - 1]?.canvas
      if (canvas) {
        const canvasRect = canvas.getBoundingClientRect()
        const canvasX = (touch.clientX - canvasRect.left) * (window.devicePixelRatio || 1)
        const canvasY = (touch.clientY - canvasRect.top) * (window.devicePixelRatio || 1)
        deleteAnnotationAt(pageNum, canvasX, canvasY)
      }
    }
    
    return
  }
  isDragging.value = false
  
  const mouseEvent = new MouseEvent('mouseup', {
    clientX: touch.clientX,
    clientY: touch.clientY
  })
  handleMouseUp(mouseEvent, pageNum)
}

// 标注创建方法
const createTextAnnotation = (pageNum: number, startX: number, startY: number, endX: number, endY: number) => {
  // 将canvas内部坐标转换为标准化坐标
  const startNormalized = convertToNormalizedCoordinates(pageNum, startX, startY)
  const endNormalized = convertToNormalizedCoordinates(pageNum, endX, endY)
  
  const minX = Math.min(startNormalized.x, endNormalized.x)
  const minY = Math.min(startNormalized.y, endNormalized.y)
  const width = Math.abs(endNormalized.x - startNormalized.x)
  const height = Math.abs(endNormalized.y - startNormalized.y)
  
  const annotation: Annotation = {
    id: Date.now() + Math.random(),
    type: currentTool.value as 'highlight' | 'underline',
    pageNum: pageNum,
    normalizedX: minX,
    normalizedY: minY,
    normalizedWidth: width,
    normalizedHeight: height,
    color: currentColor.value,
    timestamp: new Date().toISOString()
  }

  annotations.value.push(annotation)
  renderAnnotation(annotation)
}

const createTextboxAnnotation = (pageNum: number, x: number, y: number) => {
  // 将canvas内部坐标转换为标准化坐标
  const normalized = convertToNormalizedCoordinates(pageNum, x, y)
  const normalizedSize = convertSizeToNormalized(pageNum, 150, 50) // 默认文本框尺寸
  
  const annotation: Annotation = {
    id: Date.now() + Math.random(),
    type: 'text',
    pageNum: pageNum,
    normalizedX: normalized.x,
    normalizedY: normalized.y,
    normalizedWidth: normalizedSize.width,
    normalizedHeight: normalizedSize.height,
    text: '',
    color: currentColor.value,
    textColor: currentTextColor.value,
    fontSize: currentFontSize.value,
    timestamp: new Date().toISOString(),
    isEditing: true
  }

  annotations.value.push(annotation)
  renderAnnotation(annotation)
  
  // 自动聚焦到新创建的文本框
  nextTick(() => {
    const textElement = document.querySelector(`[data-annotation-id="${annotation.id}"]`) as HTMLElement
    if (textElement) {
      textElement.focus()
    }
  })
}

const createDrawingAnnotation = (pageNum: number) => {
  if (drawingPath.value.length < 2) return

  // 将绘制路径转换为标准化坐标
  const normalizedPath = drawingPath.value.map(point =>
    convertToNormalizedCoordinates(pageNum, point.x, point.y)
  )

  const annotation: Annotation = {
    id: Date.now() + Math.random(),
    type: 'draw',
    pageNum: pageNum,
    normalizedPath: normalizedPath,
    color: currentDrawColor.value,
    lineWidth: currentLineWidth.value,
    timestamp: new Date().toISOString()
  }

  annotations.value.push(annotation)
  renderAnnotation(annotation)
}

// 标注选中管理
const selectAnnotationById = (annotationId: string | number) => {
  // 取消之前选中的标注
  if (selectedAnnotationId.value) {
    const prevElement = document.querySelector(`[data-annotation-id="${selectedAnnotationId.value}"]`) as HTMLElement
    if (prevElement) {
      prevElement.classList.remove('selected')
      prevElement.style.border = prevElement.dataset.originalBorder || ''
    }
  }
  
  // 选中新的标注
  selectedAnnotationId.value = annotationId
  selectedAnnotation.value = annotations.value.find(ann => ann.id === annotationId) || null
  
  const element = document.querySelector(`[data-annotation-id="${annotationId}"]`) as HTMLElement
  if (element) {
    element.classList.add('selected')
    console.log(element)
    // 保存原始边框样式
    element.dataset.originalBorder = element.style.border
    element.style.border = '2px solid #007bff'
  }
}

const clearAnnotationSelection = () => {
  if (selectedAnnotationId.value) {
    const element = document.querySelector(`[data-annotation-id="${selectedAnnotationId.value}"]`) as HTMLElement
    if (element) {
      element.classList.remove('selected')
      element.style.border = element.dataset.originalBorder || ''
    }
  }
  selectedAnnotationId.value = null
  selectedAnnotation.value = null
}

// 标注渲染方法
const renderAnnotation = (annotation: Annotation) => {
  const pageDiv = document.querySelector(`[data-page-num="${annotation.pageNum}"]`)
  if (!pageDiv) return

  const annotationLayer = pageDiv.querySelector('.annotation-layer')
  if (!annotationLayer) return

  const annotationElement = document.createElement('div')
  annotationElement.className = `annotation-${annotation.type}`
  annotationElement.dataset.annotationId = annotation.id.toString()
  annotationElement.style.position = 'absolute'
  
  // 添加点击选中事件（除了文本框类型，因为文本框有自己的点击逻辑）
  if (annotation.type !== 'text') {
    annotationElement.addEventListener('click', (e) => {
      e.stopPropagation()
      selectAnnotationById(annotation.id)
    })
    annotationElement.style.cursor = 'pointer'
  }

  if (annotation.type === 'highlight' || annotation.type === 'underline') {
    // 将标准化坐标转换为当前显示坐标
    const position = convertFromNormalizedCoordinates(annotation.pageNum, annotation.normalizedX!, annotation.normalizedY!)
    const size = convertSizeFromNormalized(annotation.pageNum, annotation.normalizedWidth!, annotation.normalizedHeight!)
    
    annotationElement.style.left = position.x + 'px'
    annotationElement.style.top = position.y + 'px'
    annotationElement.style.width = size.width + 'px'
    annotationElement.style.height = size.height + 'px'
    
    if (annotation.type === 'highlight') {
      const rgbaColor = hexToRgba(annotation.color!, 0.3)
      annotationElement.style.backgroundColor = rgbaColor
    } else {
      annotationElement.style.borderBottom = `2px solid ${annotation.color}`
    }
  } else if (annotation.type === 'text') {
    // 将标准化坐标转换为当前显示坐标
    const position = convertFromNormalizedCoordinates(annotation.pageNum, annotation.normalizedX!, annotation.normalizedY!)
    const size = convertSizeFromNormalized(annotation.pageNum, annotation.normalizedWidth!, annotation.normalizedHeight!)
    
    annotationElement.style.left = position.x + 'px'
    annotationElement.style.top = position.y + 'px'
    annotationElement.style.width = size.width + 'px'
    annotationElement.style.height = size.height + 'px'
    annotationElement.style.backgroundColor = 'transparent'
    annotationElement.style.border = '1px dashed #ccc'
    annotationElement.style.borderRadius = '4px'
    annotationElement.style.padding = '5px'
    annotationElement.style.fontSize = (annotation.fontSize || '12') + 'px'
    annotationElement.style.color = annotation.textColor || annotation.color || '#000000'
    annotationElement.contentEditable = 'true'
    annotationElement.textContent = annotation.text || ''
    annotationElement.style.pointerEvents = 'auto'
    annotationElement.style.zIndex = '30'
    annotationElement.style.cursor = 'move'
    
    
    // 添加文本框事件
    addTextboxEvents(annotationElement, annotation)
  } else if (annotation.type === 'draw') {
    annotationElement.style.left = '0px'
    annotationElement.style.top = '0px'
    annotationElement.style.width = '100%'
    annotationElement.style.height = '100%'
    annotationElement.style.cursor = 'move';
    const canvas = document.createElement('canvas')
    const pageCanvas = pages.value[annotation.pageNum - 1]?.canvas
    if (pageCanvas && annotation.normalizedPath) {
      // 设置canvas尺寸与页面canvas一致
      canvas.width = pageCanvas.width
      canvas.height = pageCanvas.height
      canvas.style.position = 'absolute'
      canvas.style.top = '0'
      canvas.style.left = '0'
      canvas.style.width = pageCanvas.style.width
      canvas.style.height = pageCanvas.style.height
      
      const ctx = canvas.getContext('2d')!
      
      // 获取设备像素比，与页面canvas保持一致
      const pixelRatio = window.devicePixelRatio || 1
      let ratio = pixelRatio
      if (highQualityMode.value) {
        ratio = Math.max(ratio, 2.0)
      }
      
      // 缩放绘图上下文以匹配设备像素比
      ctx.scale(ratio, ratio)
      
      ctx.strokeStyle = annotation.color!
      ctx.lineWidth = parseInt(annotation.lineWidth!) || 2
      ctx.lineCap = 'round'
      ctx.lineJoin = 'round'
      ctx.beginPath()
      
      // 将标准化路径转换为当前显示坐标并绘制
      annotation.normalizedPath.forEach((normalizedPoint, index) => {
        const displayPoint = convertFromNormalizedCoordinates(annotation.pageNum, normalizedPoint.x, normalizedPoint.y)
        // 转换为canvas绘图坐标（考虑设备像素比）
        const drawX = displayPoint.x
        const drawY = displayPoint.y
        
        if (index === 0) {
          ctx.moveTo(drawX, drawY)
        } else {
          ctx.lineTo(drawX, drawY)
        }
      })
      
      ctx.stroke()
      annotationElement.appendChild(canvas)
    }
  }else if (['rectangle', 'circle', 'triangle', 'arrow'].includes(annotation.type)) {
    // 形状标注
    annotationElement.style.left = '0px'
    annotationElement.style.top = '0px'
    annotationElement.style.width = '100%'
    annotationElement.style.height = '100%'
    
    const canvas = document.createElement('canvas')
    const pageCanvas = pages.value[annotation.pageNum - 1]?.canvas
    if (pageCanvas) {
      // 设置canvas尺寸与页面canvas一致
      canvas.width = pageCanvas.width
      canvas.height = pageCanvas.height
      canvas.style.position = 'absolute'
      canvas.style.top = '0'
      canvas.style.left = '0'
      canvas.style.width = pageCanvas.style.width
      canvas.style.height = pageCanvas.style.height
      
      const ctx = canvas.getContext('2d')!
      
      // 获取设备像素比，与页面canvas保持一致
      const pixelRatio = window.devicePixelRatio || 1
      let ratio = pixelRatio
      if (highQualityMode.value) {
        ratio = Math.max(ratio, 2.0)
      }
      
      // 缩放绘图上下文以匹配设备像素比
      ctx.scale(ratio, ratio)
      
      ctx.strokeStyle = annotation.color!
      ctx.lineWidth = parseInt(annotation.lineWidth!) || 2
      ctx.lineCap = 'round'
      ctx.lineJoin = 'round'
      
      // 获取显示坐标
      const displayX = annotation.normalizedX ? annotation.normalizedX * parseFloat(pageCanvas.style.width) : 0
      const displayY = annotation.normalizedY ? annotation.normalizedY * parseFloat(pageCanvas.style.height) : 0
      const displayWidth = annotation.normalizedWidth ? annotation.normalizedWidth * parseFloat(pageCanvas.style.width) : 0
      const displayHeight = annotation.normalizedHeight ? annotation.normalizedHeight * parseFloat(pageCanvas.style.height) : 0
      
      ctx.beginPath()
      
      // 根据形状类型绘制
      switch (annotation.type) {
        case 'rectangle':
          ctx.strokeRect(displayX, displayY, displayWidth, displayHeight)
          break
          
        case 'circle':
          const centerX = annotation.centerX ? annotation.centerX * parseFloat(pageCanvas.style.width) : 0
          const centerY = annotation.centerY ? annotation.centerY * parseFloat(pageCanvas.style.height) : 0
          const radius = annotation.radius ? annotation.radius * parseFloat(pageCanvas.style.width) : 0
          ctx.arc(centerX, centerY, radius, 0, Math.PI * 2)
          ctx.stroke()
          break
          
        case 'triangle':
          ctx.moveTo(displayX + displayWidth / 2, displayY)
          ctx.lineTo(displayX + displayWidth, displayY + displayHeight)
          ctx.lineTo(displayX, displayY + displayHeight)
          ctx.closePath()
          ctx.stroke()
          break
          
        case 'arrow':
          // 绘制箭头线
          ctx.beginPath()
          ctx.moveTo(displayX, displayY)
          ctx.lineTo(displayX + displayWidth, displayY + displayHeight)
          ctx.stroke()
          
          // 绘制箭头头部
          const arrowSize = 10
          const angle = Math.atan2(displayHeight, displayWidth)
          ctx.beginPath()
          ctx.moveTo(displayX + displayWidth, displayY + displayHeight)
          ctx.lineTo(
            displayX + displayWidth - arrowSize * Math.cos(angle - Math.PI / 6),
            displayY + displayHeight - arrowSize * Math.sin(angle - Math.PI / 6)
          )
          ctx.moveTo(displayX + displayWidth, displayY + displayHeight)
          ctx.lineTo(
            displayX + displayWidth - arrowSize * Math.cos(angle + Math.PI / 6),
            displayY + displayHeight - arrowSize * Math.sin(angle + Math.PI / 6)
          )
          ctx.stroke()
          break
      }
      
      annotationElement.appendChild(canvas)
    }
  }
  annotationLayer.appendChild(annotationElement)
}

const renderAllAnnotations = () => {
  // 清除现有标注
  document.querySelectorAll('[data-annotation-id]').forEach(el => el.remove())
  document.querySelectorAll('.annotation-layer').forEach(layer => {
    layer.innerHTML = '';
  });
  // console.log('renderAllAnnotations',groupAnnotationVisible.value)
  // 重新渲染所有标注
  annotations.value.forEach(annotation => {
    renderAnnotation(annotation)
  })
}

const updateDrawingPreview = (pageNum: number) => {
  const pageDiv = document.querySelector(`[data-page-num="${pageNum}"]`)
  if (!pageDiv) return

  const preview = pageDiv.querySelector('.drawing-preview')
  if (preview) preview.remove()

  if (drawingPath.value.length < 2) return

  const annotationLayer = pageDiv.querySelector('.annotation-layer')
  if (!annotationLayer) return

  const previewElement = document.createElement('div')
  previewElement.className = 'drawing-preview'
  previewElement.style.position = 'absolute'
  previewElement.style.left = '0px'
  previewElement.style.top = '0px'
  previewElement.style.width = '100%'
  previewElement.style.height = '100%'

  const canvas = document.createElement('canvas')
  const pageCanvas = pages.value[pageNum - 1]?.canvas
  if (pageCanvas) {
    // 设置canvas尺寸与页面canvas一致
    canvas.width = pageCanvas.width
    canvas.height = pageCanvas.height
    canvas.style.position = 'absolute'
    canvas.style.top = '0'
    canvas.style.left = '0'
    canvas.style.width = pageCanvas.style.width
    canvas.style.height = pageCanvas.style.height

    const ctx = canvas.getContext('2d')!
    
    // 获取设备像素比，与页面canvas保持一致
    const pixelRatio = window.devicePixelRatio || 1
    let ratio = pixelRatio
    if (highQualityMode.value) {
      ratio = Math.max(ratio, 2.0)
    }
    
    // 缩放绘图上下文以匹配设备像素比
    ctx.scale(ratio, ratio)
    
    ctx.strokeStyle = currentDrawColor.value
    ctx.lineWidth = parseInt(currentLineWidth.value)
    ctx.lineCap = 'round'
    ctx.lineJoin = 'round'
    ctx.beginPath()

    // 直接使用canvas内部坐标进行绘制
    drawingPath.value.forEach((point, index) => {
      // 将canvas内部坐标转换为绘图坐标
      const drawX = point.x / ratio
      const drawY = point.y / ratio
      
      if (index === 0) {
        ctx.moveTo(drawX, drawY)
      } else {
        ctx.lineTo(drawX, drawY)
      }
    })

    ctx.stroke()
    previewElement.appendChild(canvas)
    annotationLayer.appendChild(previewElement)
  }
}
// 添加高亮预览函数
const updateHighlightPreview = (pageNum: number, startX: number, startY: number, endX: number, endY: number) => {
  const pageDiv = document.querySelector(`[data-page-num="${pageNum}"]`)
  if (!pageDiv) return

  // 移除之前的预览
  const preview = pageDiv.querySelector('.highlight-preview')
  if (preview) preview.remove()

  const annotationLayer = pageDiv.querySelector('.annotation-layer')
  if (!annotationLayer) return

  const previewElement = document.createElement('div')
  previewElement.className = 'highlight-preview'
  previewElement.style.position = 'absolute'
  
  // 获取canvas尺寸和比例
  const canvas = pages.value[pageNum - 1]?.canvas
  if (!canvas) return
  
  const pixelRatio = window.devicePixelRatio || 1
  let ratio = pixelRatio
  if (highQualityMode.value) {
    ratio = Math.max(ratio, 2.0)
  }
  
  // 计算预览区域的位置和大小
  const minX = Math.min(startX, endX) / ratio
  const minY = Math.min(startY, endY) / ratio
  const width = Math.abs(endX - startX) / ratio
  const height = Math.abs(endY - startY) / ratio
  
 // 形状工具预览
 if (['rectangle', 'circle', 'triangle', 'arrow'].includes(currentTool.value)) {
    const canvas = document.createElement('canvas')
    canvas.width = width * ratio
    canvas.height = height * ratio
    canvas.style.position = 'absolute'
    canvas.style.left = minX + 'px'
    canvas.style.top = minY + 'px'
    canvas.style.width = width + 'px'
    canvas.style.height = height + 'px'
    
    const ctx = canvas.getContext('2d')!
    ctx.scale(ratio, ratio)
    ctx.strokeStyle = currentDrawColor.value
    ctx.lineWidth = parseInt(currentLineWidth.value) || 2
    ctx.lineCap = 'round'
    ctx.lineJoin = 'round'
    
    // 根据不同形状绘制预览
    if (currentTool.value === 'rectangle') {
      ctx.strokeRect(0, 0, width, height)
    } else if (currentTool.value === 'circle') {
      const radius = Math.min(width, height) / 2
      ctx.beginPath()
      ctx.arc(width / 2, height / 2, radius, 0, Math.PI * 2)
      ctx.stroke()
    } else if (currentTool.value === 'triangle') {
      ctx.beginPath()
      ctx.moveTo(width / 2, 0)
      ctx.lineTo(width, height)
      ctx.lineTo(0, height)
      ctx.closePath()
      ctx.stroke()
    } else if (currentTool.value === 'arrow') {
      // 绘制箭头线
      ctx.beginPath()
      ctx.moveTo(0, 0)
      ctx.lineTo(width, height)
      ctx.stroke()
      
      // 绘制箭头头部
      const arrowSize = 10
      const angle = Math.atan2(height, width)
      ctx.beginPath()
      ctx.moveTo(width, height)
      ctx.lineTo(
        width - arrowSize * Math.cos(angle - Math.PI / 6),
        height - arrowSize * Math.sin(angle - Math.PI / 6)
      )
      ctx.moveTo(width, height)
      ctx.lineTo(
        width - arrowSize * Math.cos(angle + Math.PI / 6),
        height - arrowSize * Math.sin(angle + Math.PI / 6)
      )
      ctx.stroke()
    }
    
    previewElement.appendChild(canvas)
  } else if (currentTool.value === 'highlight') {
    previewElement.style.left = minX + 'px'
    previewElement.style.top = minY + 'px'
    previewElement.style.width = width + 'px'
    previewElement.style.height = height + 'px'
    previewElement.style.backgroundColor = hexToRgba(currentColor.value, 0.3)
  } else if (currentTool.value === 'underline') {
    previewElement.style.left = minX + 'px'
    previewElement.style.top = minY + 'px'
    previewElement.style.width = width + 'px'
    previewElement.style.borderBottom = `2px solid ${currentColor.value}`
    previewElement.style.bottom = '0'
    previewElement.style.height = '0'
  }
  
  annotationLayer.appendChild(previewElement)
}

const createShapeAnnotation = (pageNum: number, shapeType: string, startX: number, startY: number, endX: number, endY: number) => {
  // 将canvas内部坐标转换为标准化坐标
  const startNormalized = convertToNormalizedCoordinates(pageNum, startX, startY)
  const endNormalized = convertToNormalizedCoordinates(pageNum, endX, endY)
  
  const minX = Math.min(startNormalized.x, endNormalized.x)
  const minY = Math.min(startNormalized.y, endNormalized.y)
  const width = Math.abs(endNormalized.x - startNormalized.x)
  const height = Math.abs(endNormalized.y - startNormalized.y)
  
  // 计算形状的额外参数
  const shapeParams = {}
  if (shapeType === 'circle') {
    // 圆形需要半径
    shapeParams['radius'] = Math.max(width, height) / 2
    // 圆形使用中心坐标而非左上角
    shapeParams['centerX'] = minX + width / 2
    shapeParams['centerY'] = minY + height / 2
  }
  
  const annotation: Annotation = {
    id: Date.now() + Math.random(),
    type: shapeType as any,
    pageNum: pageNum,
    normalizedX: minX,
    normalizedY: minY,
    normalizedWidth: width,
    normalizedHeight: height,
    color: currentDrawColor.value,
    lineWidth: currentLineWidth.value,
    timestamp: new Date().toISOString(),
    ...shapeParams
  }

  annotations.value.push(annotation)
  renderAnnotation(annotation)
}
// 文本框事件处理
const addTextboxEvents = (element: HTMLElement, annotation: Annotation) => {
  // 添加点击选中事件
  element.addEventListener('click', (e) => {
    e.stopPropagation()
    if (!annotation.isEditing) {
      selectAnnotationById(annotation.id)
    }
  })
  
  element.addEventListener('focus', () => {
    annotation.isEditing = true
    element.style.border = '1px solid #667eea'
    selectAnnotationById(annotation.id)
  })
  
  element.addEventListener('blur', () => {
    annotation.text = element.textContent || ''
    annotation.isEditing = false
    element.style.border = '1px dashed #ccc'
    
    if (!annotation.text.trim()) {
      deleteAnnotation(annotation.id)
    }
  })
  
  element.addEventListener('input', () => {
    annotation.text = element.textContent || ''
  })
  
  // 添加拖动事件处理
  element.addEventListener('mousedown', (e: MouseEvent) => {
    // 如果正在编辑文本，不启动拖动
    if (annotation.isEditing) {
      return
    }
    
    // 阻止事件冒泡，避免触发页面的mousedown事件
    e.stopPropagation()
    
    const pageNum = annotation.pageNum
    const coords = getCanvasCoordinates(e, pageNum)
    
    isDraggingTextbox.value = true
    draggedTextbox.value = annotation
    dragStartX.value = coords.x
    dragStartY.value = coords.y
    
    // 计算鼠标相对于文本框的偏移
    const textboxPos = convertFromNormalizedCoordinates(pageNum, annotation.normalizedX!, annotation.normalizedY!)
    dragOffsetX.value = coords.x - textboxPos.x * (window.devicePixelRatio || 1)
    dragOffsetY.value = coords.y - textboxPos.y * (window.devicePixelRatio || 1)
    
    // 改变光标样式
    element.style.cursor = 'grabbing'
  })
  
  // 鼠标悬停时改变光标样式
  element.addEventListener('mouseenter', () => {
    if (!annotation.isEditing) {
      element.style.cursor = 'grab'
    }
  })
  
  element.addEventListener('mouseleave', () => {
    if (!isDraggingTextbox.value) {
      element.style.cursor = 'move'
    }
  })
}

// 标注删除方法
const deleteAnnotationAt = (pageNum: number, canvasX: number, canvasY: number) => {
  // 将canvas坐标转换为标准化坐标
  const normalized = convertToNormalizedCoordinates(pageNum, canvasX, canvasY)
  
  // 查找在该位置的标注
  const annotationToDelete = annotations.value.find(annotation => {
    if (annotation.pageNum !== pageNum) return false
    
    if (annotation.type === 'highlight' || annotation.type === 'underline' || annotation.type === 'text') {
      // 检查点击位置是否在标注范围内
      const x1 = annotation.normalizedX!
      const y1 = annotation.normalizedY!
      const x2 = x1 + annotation.normalizedWidth!
      const y2 = y1 + annotation.normalizedHeight!
      
      return normalized.x >= x1 && normalized.x <= x2 &&
             normalized.y >= y1 && normalized.y <= y2
    } else if (annotation.type === 'draw' && annotation.normalizedPath) {
      // 对于手绘标注，检查点击位置是否接近路径
      const tolerance = 0.05 // 容差值
      return annotation.normalizedPath.some(point => {
        const distance = Math.sqrt(
          Math.pow(normalized.x - point.x, 2) +
          Math.pow(normalized.y - point.y, 2)
        )
        return distance <= tolerance
      })
    } else if (['rectangle', 'circle', 'triangle', 'arrow'].includes(annotation.type)) {
      // 对于形状标注，检查点击位置是否在形状范围内
      const x1 = annotation.normalizedX!
      const y1 = annotation.normalizedY!
      const x2 = x1 + annotation.normalizedWidth!
      const y2 = y1 + annotation.normalizedHeight!
      
      return normalized.x >= x1 && normalized.x <= x2 &&
             normalized.y >= y1 && normalized.y <= y2
    }
    
    return false
  })
  
  if (annotationToDelete) {
   
    deleteAnnotation(annotationToDelete.id)
  }
}

const deleteAnnotation = (annotationId: string | number) => {
  annotations.value = annotations.value.filter(ann => ann.id != annotationId)
  console.log('annotationId',annotationId,  annotations.value );
  const element = document.querySelector(`[data-annotation-id="${annotationId}"]`)
  console.log('element',element );
  if (element) element.remove()
  renderAllAnnotations()
}

// 标注管理方法
const saveAnnotations = () => {
  if (!currentFileName.value) {
    alert('请先加载 PDF 文件')
    return
  }

  const data = {
    fileName: currentFileName.value,
    annotations: annotations.value,
    timestamp: new Date().toISOString()
  }

  localStorage.setItem(`pdf_annotations_${currentFileName.value}`, JSON.stringify(data))
  alert('标注已保存到本地')
}

const loadAnnotations = () => {
  if (!currentFileName.value) return
  if (annotations.value) {
    try {
      renderAllAnnotations()
      console.log('标注已从本地加载')
    } catch (error) {
      console.error('加载标注失败:', error)
    }
  }
}

const onShowAnnotations=()=>{
  isShowAnnotation.value = !isShowAnnotation.value;
  if(isShowAnnotation.value){
    renderAllAnnotations()
  }else{
    clearAllAnnotations()
  }
}
const clearAllAnnotations = () => {
  document.querySelectorAll('[data-annotation-id]').forEach(el => el.remove())
}

// 调用父组件方法提交
const submitAnnotations = async () => {
  
  emit('saveAnnotations', annotations.value)
}

// 标注列表方法
const toggleAnnotationList = () => {
  showAnnotationList.value = !showAnnotationList.value
}

const groupAnnotationVisible = ref([]);



const expandedGroups = ref<Record<string, boolean>>({});

// 切换分组展开/折叠状态
const toggleGroup = (doctorId: string | number) => {
  // 初始化状态（默认展开）
  if (expandedGroups.value[doctorId] === undefined) {
    expandedGroups.value[doctorId] = true;
  } else {
    expandedGroups.value[doctorId] = !expandedGroups.value[doctorId];
  }
};

const selectAnnotation = (annotation: Annotation) => {
  // 使用统一的选中方法
  selectAnnotationById(annotation.id)
  scrollToPage(annotation.pageNum)
}

const getAnnotationTypeName = (type: string) => {
  const typeMap: Record<string, string> = {
    highlight: '高亮',
    underline: '下划线',
    text: '文本框',
    draw: '绘制',
    triangle: '三角形',
    rectangle: '矩形',
    circle: '圆形',
    arrow: '箭头',
    delete: '删除'
  }
  return typeMap[type] || type
}

const getToolIcon = (type: string) => {
  const iconMap: Record<string, string> = {
    highlight: 'ri-mark-pen-line',
    underline: 'ri-underline',
    text: 'ri-text',
    draw: 'ri-pencil-line',
    delete: 'ri-eraser-line',
    triangle: 'ri-triangle-line',
  }
  return iconMap[type] || 'ri-tools-line'
}

const formatTime = (timestamp: string) => {
  return new Date(timestamp).toLocaleString()
}

// 工具函数
const hexToRgba = (hex: string, alpha: number) => {
  hex = hex.replace('#', '')
  const r = parseInt(hex.substr(0, 2), 16)
  const g = parseInt(hex.substr(2, 2), 16)
  const b = parseInt(hex.substr(4, 2), 16)
  return `rgba(${r}, ${g}, ${b}, ${alpha})`
}

// 键盘事件处理
const handleKeydown = (event: KeyboardEvent) => {
  if (!props.showToolbar) return
  
  switch (event.key) {
    case 'ArrowLeft':
      event.preventDefault()
      previousPage()
      break
    case 'ArrowRight':
      event.preventDefault()
      nextPage()
      break
    case '+':
    case '=':
      event.preventDefault()
      zoomIn()
      break
    case '-':
      event.preventDefault()
      zoomOut()
      break
    case '0':
      if (event.ctrlKey) {
        event.preventDefault()
        resetZoom()
      }
      break
    case 'f':
    case 'F11':
      if (event.ctrlKey || event.key === 'F11') {
        event.preventDefault()
        toggleFullscreen()
      }
      break
  }
}

// 全屏状态监听
const handleFullscreenChange = () => {
  isFullscreen.value = !!document.fullscreenElement
}

// 全局鼠标事件处理（用于文本框拖动）
const handleGlobalMouseUp = () => {
  if (isDraggingTextbox.value) {
    isDraggingTextbox.value = false
    if (draggedTextbox.value) {
      // 恢复光标样式
      const textboxElement = document.querySelector(`[data-annotation-id="${draggedTextbox.value.id}"]`) as HTMLElement
      if (textboxElement) {
        textboxElement.style.cursor = 'move'
      }
    }
    draggedTextbox.value = null
  }
}

// 添加拖动开始处理
const handleDragStart = (e: MouseEvent) => {
  // 如果正在使用标注工具，不启用拖动
  if (currentTool.value) return
  
  isDragging.value = true
  dragStartX.value = e.clientX
  dragStartY.value = e.clientY
  
  // 更改鼠标样式
  if (allPagesContainer.value) {
    allPagesContainer.value.style.cursor = 'grabbing'
  }
  
  e.preventDefault()
}

// 添加拖动结束处理
const handleDragEnd = () => {
  isDragging.value = false
  
  // 恢复鼠标样式
  if (allPagesContainer.value) {
    allPagesContainer.value.style.cursor = 'grab'
  }
}

const handleGlobalMouseMove = (e: MouseEvent) => {
  if (isDraggingTextbox.value && draggedTextbox.value) {
    // 找到对应的页面元素
    const pageElement = document.querySelector(`[data-page-num="${draggedTextbox.value.pageNum}"]`)
    if (pageElement) {
      const canvas = pageElement.querySelector('.pdf-canvas') as HTMLCanvasElement
      if (canvas) {
        const canvasRect = canvas.getBoundingClientRect()
        const canvasX = (e.clientX - canvasRect.left) * (window.devicePixelRatio || 1)
        const canvasY = (e.clientY - canvasRect.top) * (window.devicePixelRatio || 1)
        
        const normalized = convertToNormalizedCoordinates(draggedTextbox.value.pageNum, canvasX - dragOffsetX.value, canvasY - dragOffsetY.value)
        
        // 更新文本框位置
        draggedTextbox.value.normalizedX = Math.max(0, Math.min(1 - draggedTextbox.value.normalizedWidth!, normalized.x))
        draggedTextbox.value.normalizedY = Math.max(0, Math.min(1 - draggedTextbox.value.normalizedHeight!, normalized.y))
        
        // 重新渲染该文本框
        const textboxElement = document.querySelector(`[data-annotation-id="${draggedTextbox.value.id}"]`) as HTMLElement
        if (textboxElement) {
          const position = convertFromNormalizedCoordinates(draggedTextbox.value.pageNum, draggedTextbox.value.normalizedX!, draggedTextbox.value.normalizedY!)
          textboxElement.style.left = position.x + 'px'
          textboxElement.style.top = position.y + 'px'
        }
      }
    }
  }
  // 添加PDF内容拖动逻辑
  if (isDragging.value) {
    const deltaX = e.clientX - dragStartX.value
    const deltaY = e.clientY - dragStartY.value
    
    scrollOffsetX.value += deltaX
    scrollOffsetY.value += deltaY
    
    dragStartX.value = e.clientX
    dragStartY.value = e.clientY
    
    // 应用滚动偏移
    if (allPagesContainer.value) {
      allPagesContainer.value.style.transform = `translate(${scrollOffsetX.value}px, ${scrollOffsetY.value}px)`
    }
  }
}

// 全局点击事件处理（用于文本框失焦）
const handleGlobalClick = (e: MouseEvent) => {
  const target = e.target as HTMLElement
  
  // 检查是否点击了文本框
  const isTextboxClick = target.closest('.annotation-text')
  
  // 如果没有点击文本框，让所有正在编辑的文本框失焦
  if (!isTextboxClick) {
    const editingTextboxes = document.querySelectorAll('.annotation-text[contenteditable="true"]:focus')
    editingTextboxes.forEach(textbox => {
      (textbox as HTMLElement).blur()
    })
  }
}
const emit = defineEmits(['saveAnnotations'])

// 监听 annotationData 变化
watch(() => props.annotationData, (newData:any) => {
    console.log("newData",newData)
  annotations.value = newData
//   formatAnnotationData(newData);
  // 重新渲染标注
  loadAnnotations();
}, { immediate: true })

// 监听props变化
watch(() => props.src, () => {
  if (props.src) {
    loadPDF()
  }
}, { immediate: true })

// 生命周期
onMounted(() => {
  document.addEventListener('keydown', handleKeydown)
  document.addEventListener('fullscreenchange', handleFullscreenChange)
  document.addEventListener('mouseup', handleGlobalMouseUp)
  document.addEventListener('mousemove', handleGlobalMouseMove)
  // document.addEventListener('click', handleGlobalClick)
})

onUnmounted(() => {
  document.removeEventListener('keydown', handleKeydown)
  document.removeEventListener('fullscreenchange', handleFullscreenChange)
  document.removeEventListener('mouseup', handleGlobalMouseUp)
  document.removeEventListener('mousemove', handleGlobalMouseMove)
  document.removeEventListener('click', handleGlobalClick)
})

// 暴露方法给父组件
defineExpose({
  loadPDF,
  previousPage,
  nextPage,
  goToPage,
  zoomIn,
  zoomOut,
  resetZoom,
  toggleFullscreen,
  saveAnnotations,
  loadAnnotations,
  clearAllAnnotations,
  currentPage: () => currentPage.value,
  totalPages: () => totalPages.value,
  scale: () => scale.value,
  annotations: () => annotations.value
})
</script>

<style scoped>
.pdf-viewer-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #f5f5f5;
  position: relative;
}

.pdf-toolbar {
  background: #38383d;
  border-bottom: 1px solid #585151;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  flex-shrink: 0;
  position: relative;
}

.toolbar-toggle {
  display: none;
  width: 100%;
  padding: 12px;
  background: #fff;
  border: none;
  border-bottom: 1px solid #e0e0e0;
  cursor: pointer;
  font-size: 14px;
  color: #666;
  transition: background-color 0.2s;
}

.toolbar-toggle:hover {
  background: #f8f9fa;
}

.toolbar-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 16px;
  gap: 16px;
  background-color: #38383d;
}

.toolbar-section {
  display: flex;
  align-items: center;
  gap: 8px;
}

.toolbar-navigation {
  flex-shrink: 0;
}

.toolbar-zoom {
  flex: 1;
  justify-content: center;
}

.toolbar-actions {
  flex-shrink: 0;
}

.toolbar-btn {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  /* border: 1px solid #d0d0d0; */
  color: #fff;
  background: transparent;
  border-radius: 6px;
  cursor: pointer;  
  font-size: 14px;
  transition: all 0.2s;
  /* min-height: 36px; */
}

.toolbar-btn:hover:not(:disabled) {
  background: #59595E;
  border-color: #fff;
  transform: translateY(-1px);
}

.toolbar-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.toolbar-btn.active {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

.btn-text {
  font-size: 14px;
}

.page-info {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 14px;
  color: #666;
}

.page-total {
  color: #999;
  font-size: 14px;
}

.zoom-info {
  font-size: 14px;
  color: #666;
  min-width: 60px;
  text-align: center;
  font-weight: 500;
}

.page-input-inline {
  width: 50px;
  padding: 4px 6px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  text-align: center;
  font-size: 14px;
}

/* 标注工具栏样式 */
.annotation-toolbar {
  background: #f8f9fa;
  /* border-bottom: 1px solid #e0e0e0; */
  flex-shrink: 0;
}

.annotation-toolbar-scroll {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 16px;
  gap: 16px;
  position: relative;
  /* overflow-x: auto; */
  scrollbar-width: none;
  -ms-overflow-style: none;
  background-color: #38383d;
}

.annotation-toolbar-scroll::-webkit-scrollbar {
  display: none;
}

.annotation-tools {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.tool-wrapper {
  position: relative;
}

.tool-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 8px 12px;
  /* border: 1px solid #d0d0d0; */
  color: #fff;
  background: transparent;
  border-radius: 6px;
  cursor: pointer;
  font-size: 20px;
  /* transition: all 0.2s; */
  /* min-width: 60px;
  min-height: 50px; */
}

.tool-btn:hover {
  background: #59595E;
  border-color: #fff;
  transform: translateY(-1px);
}

.tool-btn.active {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

.tool-name {
  font-size: 10px;
  text-align: center;
  line-height: 1.2;
}

.tool-settings-panel {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-shrink: 0;
}

.tool-settings {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 12px;
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.setting-group {
  display: flex;
  align-items: center;
  gap: 6px;
}

.setting-group label {
  font-size: 12px;
  color: #666;
  white-space: nowrap;
}

.color-picker {
  width: 32px;
  height: 28px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  cursor: pointer;
}

.font-size-select,
.line-width-select {
  padding: 4px 8px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  font-size: 14px;
  min-width: 60px;
}

.annotation-actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.action-btn {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 8px 12px;
  border: 1px solid #d0d0d0;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s;
  min-height: 36px;
}

.action-text {
  font-size: 11px;
}

.save-btn {
  background: #28a745;
  color: white;
  border-color: #28a745;
}

.load-btn {
  background: #17a2b8;
  color: white;
  border-color: #17a2b8;
}

.clear-btn {
  background: #ffc107;
  color: #212529;
  border-color: #ffc107;
}

.submit-btn {
  background: #007bff;
  color: white;
  border-color: #007bff;
}

.action-btn:hover {
  opacity: 0.8;
}

/* 悬浮工具设置面板 */
.floating-tool-settings {
  position: absolute;
  top: 120px;
  left: 50%;
  transform: translateX(-50%);
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  z-index: 1001;
  padding: 12px 16px;
  animation: slideDown 0.3s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

.floating-tool-settings .tool-settings {
  display: flex;
  align-items: center;
  gap: 12px;
  margin: 0;
  padding: 0;
  background: transparent;
  border: none;
  box-shadow: none;
}

.floating-tool-settings .setting-group {
  display: flex;
  align-items: center;
  gap: 6px;
}

.floating-tool-settings .setting-group label {
  font-size: 12px;
  color: #666;
  white-space: nowrap;
  font-weight: 500;
}

/* 标注选中样式 */
.annotation-highlight.selected,
.annotation-underline.selected,
.annotation-text.selected,
.annotation-draw.selected {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

/* 主内容区域 */
.pdf-main-content {
  display: flex;
  flex: 1;
  overflow: hidden
;
}

.pdf-content {
  flex: 1;
  position: relative;
  overflow: auto;
  background: #2a2a2e;
}

.pdf-content.fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: 9999;
}

.pdf-iframe {
  width: 100%;
  height: 100%;
  border: none;
}

.pdf-canvas-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  /* align-items: flex-start; */
  /* padding: 20px; */
  overflow: auto;
  height: 100%;
  cursor: grab;
  user-select: none;
}

.all-pages-container {
  display: flex;
  flex-direction: column;
  gap: 15px;
  padding: 10px 0;

  position: relative;
  will-change: transform;
  transition: transform 0.2s ease;
}

.pdf-page {
  position: relative;
  margin: 0 auto ;
  background: white;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
}

.pdf-canvas {
  background: white;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  transform-origin: center top;
  transition: transform 0.2s;
  margin: 0 auto;
}

.annotation-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 10;
}

.interaction-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  cursor: crosshair;
  z-index: 20;
}

/* 标注样式 */
.annotation-highlight {
  pointer-events: none;
}

.annotation-underline {
  pointer-events: none;
}

.annotation-text {
  pointer-events: auto;
  outline: none;
  resize: both;
  overflow: auto;
  min-width: 50px;
  min-height: 20px;
  cursor: move;
}

.annotation-text:focus {
  border: 1px solid #667eea !important;
}

.annotation-text[data-placeholder]:empty::before {
  content: attr(data-placeholder);
  color: #999;
}

.annotation-draw {
  pointer-events: none;
}

.loading-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 200px;
  color: #fff;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #007bff;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 16px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 200px;
  color: #fff;
}

.zoom-select{
  /* padding: 4px 8px;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  font-size: 14px;
  color: #666;
  background-color: #fff;
  cursor: pointer;
  min-width: 100px; */

  -webkit-appearance:none;
    -moz-appearance:none;
    appearance:none;
    width:inherit;
    min-width:inherit;
    height:28px;
    font:message-box;
    font-size:12px;
    color:#fff;
    margin:0;
    padding-block:1px 2px;
    padding-inline:6px 38px;
    border:none;
    outline:none;
    background-color:#4a4a4f;
}
.zoom-select-box{ 
  position: relative; 
}

.zoom-select-box::after{
  content: "";
  position: absolute;
  right: 12px; /* 调整箭头位置 */
  top: 50%;
  transform: translateY(-50%);
  width: 0;
  height: 0;
  border-left: 4px solid transparent;
  border-right: 4px solid transparent;
  border-top: 6px solid white; /* 创建向下箭头 */
  pointer-events: none; /* 避免影响点击事件 */
}


.error-message {
  color: #ff6b6b;
  margin-bottom: 16px;
  text-align: center;
}

.retry-btn {
  padding: 8px 16px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.retry-btn:hover {
  background: #0056b3;
}


/* 抽屉遮罩层 */
.annotation-drawer-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 999;
  opacity: 0;
  visibility: hidden;
  transition: all 0.3s ease;
}

.annotation-drawer-overlay.show {
  opacity: 1;
  visibility: visible;
}

/* 抽屉容器 */
.annotation-drawer {
  position: fixed;
  top: 0; /* 顶部对齐 */
  right: 0; /* 右侧对齐 */
  bottom: auto; /* 移除底部定位 */
  left: auto; /* 移除左侧定位 */
  width: 25vw; /* 使用宽度代替高度 */
  height: 100%; /* 高度占满视口 */
  background: white;
  border-radius: 16px 0 0 16px; /* 调整圆角 */
  box-shadow: -4px 0 20px rgba(0, 0, 0, 0.15); /* 调整阴影方向 */
  z-index: 1000;
  transform: translateX(100%); /* 初始状态移出可视区域 */
  transition: transform 0.3s ease;
  display: flex;
  flex-direction: column;
}

.annotation-drawer.show {
  transform: translateX(0); /* 显示时滑入可视区域 */
}

/* 抽屉拖拽手柄 */
.drawer-handle {
  position: absolute;
  top: 50%;
  left: -20px; /* 手柄显示在左侧 */
  transform: translateY(-50%);
  flex-direction: column;
  padding: 12px 8px;
  background: #f8f9fa;
  border-radius: 16px 0 0 16px;
  box-shadow: -2px 0 8px rgba(0, 0, 0, 0.1);
}

.handle-bar {
  width: 4px;
  height: 40px;
  background: #d0d0d0;
  border-radius: 2px;
  transition: background-color 0.2s;
}


.drawer-handle:hover .handle-bar {
  background: #b0b0b0;
}

/* 抽屉头部 */
.drawer-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #e0e0e0;
  background: #f8f9fa;
}

.drawer-header h3 {
  margin: 0;
  font-size: 18px;
  color: #333;
  font-weight: 600;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #666;
  padding: 4px;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: all 0.2s;
}

.close-btn:hover {
  color: #333;
  background: #e9ecef;
}

.drawer-footer {
  display: flex;
  justify-content:center;
  align-items: center;
  padding: 15px 0px;
  border-bottom: 1px solid #e0e0e0;
  background: #f8f9fa;
}
.drawer-footer  .submit-btn{

  width: 80%;
  border-radius: 20px;
  padding: 10px 0;
}

/* 抽屉内容区域 */
.drawer-content {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.annotation-list {
  flex: 1;
  overflow-y: auto;
  padding: 16px 20px;
}

.annotation-item {
  padding: 16px;
  border: 1px solid #e0e0e0;
  border-radius: 12px;
  margin-bottom: 12px;
  cursor: pointer;
  transition: all 0.2s;
  background: white;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.annotation-item:hover {
  border-color: #667eea;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.15);
  transform: translateY(-1px);
}

.annotation-item.selected {
  border-color: #667eea;
  background: #f0f2ff;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
}

.annotation-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.annotation-type {
  font-weight: 600;
  color: #667eea;
  font-size: 14px;
  background: #f0f2ff;
  padding: 4px 8px;
  border-radius: 12px;
}
.annotation-doctorName{
  font-size: 14px;
  color: #666;
}

.annotation-page {
  font-size: 12px;
  color: #666;
  background: #f8f9fa;
  padding: 2px 6px;
  border-radius: 8px;
}

.annotation-content {
  font-size: 14px;
  color: #333;
  margin-bottom: 8px;
  word-break: break-word;
  line-height: 1.4;
}

.annotation-time {
  font-size: 12px;
  color: #999;
  text-align: left;
}

.delete-annotation-btn {
  background: #fff;
  color: #dc3545;
  /* border: 1px solid #dc3545; */
  padding: 6px;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.delete-annotation-btn:hover {
  background: #dc3545;
  color: white;
  /* transform: scale(1.1); */
}

/* 标注列表切换按钮 */
.annotation-list-toggle {
  position: fixed;
  right: 20px;
  top: 50%;
  transform: translateY(-50%);
  background: #667eea;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 10px 15px;
  cursor: pointer;
  font-size: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  transition: all 0.2s;
  z-index: 1000;
}

.annotation-list-toggle:hover {
  background: #5a6fd8;
  transform: translateY(-50%) scale(1.05);
}

.annotation-list-toggle.active {
  background: #28a745;
}

/* 手机端样式 */
.mobile-only {
  display: none;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .mobile-only {
    display: flex;
  }
  
  .pdf-toolbar {
    padding: 0;
  }
  
  .toolbar-content {
    padding: 8px 12px;
    gap: 8px;
    overflow-x: auto;
    scrollbar-width: none;
    -ms-overflow-style: none;
  
  }
  
  .toolbar-content::-webkit-scrollbar {
    display: none;
  }
  .annotation-type , .annotation-doctorName{
    font-size: 13px;
  }
  .annotation-page{
    font-size: 11px;
  }
  /* .toolbar-content:not(.expanded) {
    display: none;
  } */
  
  .toolbar-section {
    flex-shrink: 0;
    gap: 6px;
  }
  
  .toolbar-btn {
    padding: 3px 5px;
    font-size: 18px;
    /* min-height: 40px;
    min-width: 40px; */
  }
  
  .btn-text {
    display: none;
  }
  
  .page-input-inline {
    width: 40px;
    font-size: 12px;
  }
  
  .zoom-info {
    font-size: 12px;
    min-width: 50px;
  }

  /* 标注工具栏手机端样式 */
  .annotation-toolbar-scroll {
    padding: 8px 12px;
    gap: 12px;

  }
  
  .annotation-tools {
    gap: 6px;
  }
  
  .tool-btn {
    /* min-width: 50px;
    min-height: 44px; */
    font-size: 15px;
    padding: 6px 8px;
  }
  
  .tool-name {
    font-size: 9px;
  }
  
  .tool-settings-panel {
    gap: 8px;
  }
  
  .tool-settings {
    padding: 6px 8px;
    gap: 8px;
  }
  
  .setting-group {
    gap: 4px;
  }
  
  .color-picker {
    width: 28px;
    height: 24px;
  }
  
  .font-size-select,
  .line-width-select {
    min-width: 50px;
    font-size: 11px;
  }
  
  .annotation-actions {
    gap: 6px;
  }
  
  .action-btn {
    min-height: 20px;
    padding: 4px 10px;
  }
  
  .action-text {
    display: none;
  }

  /* 抽屉在手机端的调整 */
  .annotation-drawer {
    width: 52%;
    /* height: 75vh; */
    /* max-height: calc(100vh - 60px); */
  }
  
  .drawer-header {
    padding: 12px 16px;
  }
  
  .drawer-header h3 {
    font-size: 16px;
  }
  
  .annotation-list {
    padding: 12px 16px;
  }
  
  .annotation-item {
    padding: 6px;
    margin-bottom: 10px;
  }
  
  .annotation-header {
    margin-bottom: 6px;
  }
  
  .annotation-content {
    font-size: 13px;
    margin-bottom: 6px;
  }

  /* 标注列表切换按钮 */
  .annotation-list-toggle {
    bottom: 20px;
    right: 20px;
    top: auto;
    transform: none;
    padding: 12px 16px;
    font-size: 14px;
    border-radius: 25px;
    box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  }
  
  .annotation-list-toggle:hover {
    transform: scale(1.05);
  }
}

/* 超小屏幕适配 */
@media (max-width: 480px) {
  .toolbar-content {
    padding: 6px 8px;
    gap: 6px;
  }
  
  .toolbar-btn {
    /* padding: 6px 8px; */
    /* min-width: 36px;
    min-height: 36px; */
  }
  
  .annotation-toolbar-scroll {
    padding: 6px 8px;
    gap: 8px;
  }
  
  .tool-btn {
    padding: 4px 6px;
  }
  
  .annotation-drawer {
    /* height: 80vh; */
  }
  
  .drawer-header {
    padding: 10px 12px;
  }
  
  .annotation-list {
    padding: 8px 12px;
  }
  
  .annotation-item {
    padding: 6px;
    margin-bottom: 8px;
  }
}

/* 打印样式 */
@media print {
  .pdf-toolbar,
  .annotation-toolbar,
  .annotation-sidebar,
  .annotation-list-toggle {
    display: none !important;
  }

  .pdf-content {
    background: white !important;
  }

  .pdf-canvas {
    box-shadow: none !important;
  }
}
</style>