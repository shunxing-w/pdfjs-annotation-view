# 构建功能强大的PDF标注查看器：基于Vue 3和PDF.js的实现

## 项目背景

在日常工作和学习中，我们经常需要查看和标注PDF文档。虽然市面上有很多PDF阅读器，但大多数要么功能受限，要么收费昂贵。为了解决这个问题，我决定开发一个开源、功能完整的PDF标注查看器，让用户可以免费使用并自定义功能。

## 功能介绍

这个PDF标注查看器具有以下核心功能：

### PDF查看功能
- 页面导航：上一页、下一页、页码输入
- 缩放控制：放大、缩小、自动缩放、实际大小、适合页面、适合页宽
- 旋转控制：左转、右转
- 全屏模式

### 标注功能
- 高亮：可自定义颜色
- 下划线：可自定义颜色
- 文本框：可自定义颜色和字体大小
- 绘图：可自定义颜色和线条粗细
- 形状标注：矩形、圆形、三角形、箭头
- 标注删除和编辑
- 标注列表查看和管理
- 标注保存到本地存储

## 技术栈选择

为什么选择以下技术栈？

- **Vue 3**：作为前端框架，Vue 3提供了Composition API，使代码组织更加灵活，性能也有显著提升。
- **Vue Router**：用于管理页面路由，实现首页和PDF查看页之间的导航。
- **PDF.js**：Mozilla开发的开源PDF查看器库，功能强大，支持多种PDF渲染方式。
- **Vite**：作为构建工具，Vite提供了快速的开发体验和优化的生产构建。

## 项目结构

```
pdfjs-annotation-view/
├── .gitignore
├── .vscode/
│   └── extensions.json
├── README.md
├── BLOG_POST.md
├── index.html
├── package-lock.json
├── package.json
├── public/
│   ├── EUPL.pdf
│   ├── pdfWebViewer/
│   │   ├── LICENSE
│   │   ├── build/
│   │   └── web/
│   └── vite.svg
├── src/
│   ├── App.vue
│   ├── assets/
│   │   └── vue.svg
│   ├── components/
│   │   └── PdfViewer.vue
│   ├── main.js
│   ├── router/
│   │   └── index.js
│   ├── style.css
│   └── views/
│       ├── HomeView.vue
│       └── pdf.vue
└── vite.config.js
```

## 核心实现

### 1. PDF渲染实现

项目使用PDF.js库来渲染PDF文件，支持两种渲染模式：iframe和canvas。默认使用canvas模式，因为它提供了更好的性能和标注交互体验。

```javascript
// 配置PDF.js worker
import * as pdfjsLib from 'pdfjs-dist'
import PdfWorker from "pdfjs-dist/build/pdf.worker.min.mjs?url"

dpdfjsLib.GlobalWorkerOptions.workerSrc = PdfWorker
```

### 2. 标注系统设计

标注系统是项目的核心，我采用了以下设计方案：

- 每种标注类型（高亮、文本框、绘图等）都有统一的接口定义
- 使用canvas绘制标注，确保高性能和跨浏览器兼容性
- 标注数据结构包含类型、位置、样式和内容等信息
- 使用本地存储保存标注数据

```typescript
// 标注接口定义
export interface Annotation {
  id: string | number
  type: 'highlight' | 'underline' | 'text' | 'draw' | 'rectangle' | 'circle' | 'triangle' | 'arrow'
  pageNum: number
  position: { x: number; y: number; width?: number; height?: number }
  points?: { x: number; y: number }[]
  text?: string
  color: string
  lineWidth?: number
  fontSize?: number
  timestamp: number
}
```

### 3. 工具栏和交互设计

为了提供良好的用户体验，我设计了直观的工具栏和交互方式：

- 顶部工具栏用于PDF查看控制
- 底部工具栏用于标注工具选择
- 工具设置面板用于自定义标注样式
- 标注列表抽屉用于查看和管理所有标注

## 关键代码解析

### PDF加载和渲染

```javascript
// 加载PDF文档
const loadPdf = async () => {
  try {
    setLoading(true)
    setError('')

    const loadingTask = pdfjsLib.getDocument(src)
    const pdf = await loadingTask.promise
    setPdfDoc(pdf)
    setTotalPages(pdf.numPages)

    // 加载初始页面
    await renderPage(initialPage)
    setCurrentPage(initialPage)
    setPageInput(initialPage)
  } catch (err) {
    setError(`加载PDF失败: ${err instanceof Error ? err.message : String(err)}`)
    console.error('PDF加载错误:', err)
  } finally {
    setLoading(false)
  }
}
```

### 标注绘制

```javascript
// 绘制高亮标注
drawHighlight = (ctx, annotation) => {
  const { position, color } = annotation
  ctx.save()
  ctx.fillStyle = `${color}40` // 添加透明度
  ctx.fillRect(position.x, position.y, position.width, position.height)
  ctx.restore()
}

// 绘制文本框标注
drawTextAnnotation = (ctx, annotation) => {
  const { position, text, color, fontSize = 16 } = annotation
  ctx.save()
  ctx.fillStyle = `${color}40` // 背景色
  ctx.fillRect(position.x, position.y, position.width, position.height)
  ctx.fillStyle = color // 文本色
  ctx.font = `${fontSize}px Arial`
  ctx.fillText(text || '', position.x + 5, position.y + fontSize)
  ctx.restore()
}
```

### 标注保存和加载

```javascript
// 保存标注
export const saveAnnotations = (annotations) => {
  try {
    localStorage.setItem('annotations', JSON.stringify(annotations))
    return true
  } catch (error) {
    console.error('保存标注失败:', error)
    return false
  }
}

// 加载标注
export const loadAnnotations = () => {
  try {
    const annotations = localStorage.getItem('annotations')
    return annotations ? JSON.parse(annotations) : []
  } catch (error) {
    console.error('加载标注失败:', error)
    return []
  }
}
```

## 使用方法

1. 克隆项目并安装依赖

```bash
git clone <项目地址>
cd pdfjs-annotation-view
npm install
```

2. 运行开发服务器

```bash
npm run dev
```

3. 在浏览器中访问 `http://localhost:5173`，点击"前往pdf预览页面"按钮

4. 使用工具栏中的标注工具进行标注操作

5. 点击"保存"按钮保存标注

## 遇到的问题及解决方案

### 1. PDF渲染性能问题

**问题**：大型PDF文件渲染缓慢，占用内存高。

**解决方案**：
- 实现按需渲染，只渲染当前可见页面
- 使用虚拟滚动技术，只加载视口内的页面
- 优化canvas绘制，避免不必要的重绘

### 2. 标注定位不准确

**问题**：在不同缩放级别下，标注位置可能偏移。

**解决方案**：
- 使用相对坐标而非绝对坐标存储标注位置
- 每次缩放或滚动后，重新计算标注位置
- 实现坐标转换函数，确保不同缩放级别下的一致性

### 3. 移动端兼容性问题

**问题**：在移动设备上触摸交互不流畅。

**解决方案**：
- 添加触摸事件处理，支持移动端交互
- 优化工具栏布局，适应小屏幕设备
- 实现手势缩放和滑动翻页功能

## 总结与展望

这个PDF标注查看器项目已经实现了基本的PDF查看和标注功能，但还有很多可以改进的地方：

1. 添加用户认证和云存储，实现标注数据的跨设备同步
2. 支持更多标注类型和样式自定义选项
3. 优化移动端体验，添加更多手势控制
4. 实现PDF文本搜索功能
5. 支持PDF表单填写和签名功能

如果你对这个项目感兴趣，欢迎贡献代码或提出建议。让我们一起打造一个功能强大、免费开源的PDF标注查看器！

## 项目地址

[GitHub仓库链接]（待添加）