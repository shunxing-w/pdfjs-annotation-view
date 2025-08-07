# PDF.js 标注查看器

`pdfjs-annotation-view` 是一个基于 PDF.js 和 Vue 3 开发的 PDF 查看器，支持多种标注功能，如高亮、下划线、文本框、绘图等。

## 项目功能

- **PDF 查看功能**
  - 页面导航（上一页、下一页、页码输入）
  - 缩放控制（放大、缩小、自动缩放、实际大小、适合页面、适合页宽等）
  - 旋转控制（左转、右转）
  - 全屏模式

- **标注功能**
  - 高亮（可自定义颜色）
  - 下划线（可自定义颜色）
  - 文本框（可自定义颜色和字体大小）
  - 绘图（可自定义颜色和线条粗细）
  - 形状标注（矩形、圆形、三角形、箭头）
  - 标注删除
  - 标注列表查看和管理
  - 标注保存到本地存储

## 技术栈

- Vue 3.5.17
- Vue Router 4.5.1
- PDF.js 4.8.69
- Vite 7.0.4

## 项目结构

```
pdfjs-annotation-view/
├── .gitignore
├── .vscode/
│   └── extensions.json
├── README.md
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

## 核心组件

### PdfViewer.vue

这是项目的核心组件，负责PDF的渲染和标注功能。它支持两种渲染模式：iframe和canvas（默认使用canvas）。

主要属性：
- `src`: PDF文件路径
- `showToolbar`: 是否显示工具栏
- `showAnnotationTools`: 是否显示标注工具栏
- `viewerMode`: 查看模式（'iframe' 或 'canvas'）
- `initialPage`: 初始页码
- `initialScale`: 初始缩放比例
- `annotationData`: 标注数据

主要事件：
- `saveAnnotations`: 保存标注时触发

## 页面路由

- `/`: 首页
- `/pdf`: PDF查看页面

## 安装和运行

1. 克隆项目

```bash
git clone <项目地址>
cd pdfjs-annotation-view
```

2. 安装依赖

```bash
npm install
```

3. 运行开发服务器

```bash
npm run dev
```

4. 构建生产版本

```bash
npm run build
```

5. 预览生产版本

```bash
npm run preview
```

## 使用说明

1. 打开PDF查看页面

在浏览器中访问 `http://localhost:5173/pdf` 即可打开PDF查看器。

2. 使用标注工具

在标注工具栏中选择所需的标注工具，然后在PDF页面上进行标注操作。标注工具包括高亮、下划线、文本框、绘图等。

3. 保存标注

点击标注工具栏中的"保存"按钮，标注将保存到本地存储中。

4. 查看标注列表

点击标注工具栏中的"标注列表"按钮，可以查看所有标注，并进行编辑或删除操作。

## 配置

### 自定义PDF文件

在 `src/views/pdf.vue` 文件中，可以修改 `pdfSrc` 变量来指定要加载的PDF文件路径。

```javascript
// src/views/pdf.vue
const pdfSrc ='http://localhost:5173/EUPL.pdf';
```

### 自定义初始设置

在 `src/views/pdf.vue` 文件中，可以修改 `initialPage` 和 `initialScale` 属性来设置初始页码和缩放比例。

```html
<!-- src/views/pdf.vue -->
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
```

## 注意事项

1. 本项目使用本地存储保存标注数据，刷新页面后标注数据不会丢失，但仅在当前浏览器中有效。

2. 为了获得最佳体验，请使用现代浏览器。

3. 大型PDF文件可能需要较长时间加载，请耐心等待。