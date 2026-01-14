# OpenScreen 项目结构与流程详解

## 目录

1. [项目概述](#项目概述)
2. [技术栈](#技术栈)
3. [项目目录结构](#项目目录结构)
4. [录制流程](#录制流程)
5. [编辑流程](#编辑流程)
6. [导出流程](#导出流程)
7. [组件层次结构](#组件层次结构)
8. [IPC 通信](#ipc-通信)
9. [状态管理](#状态管理)
10. [核心数据结构](#核心数据结构)
11. [完整工作流](#完整工作流)

---

## 项目概述

OpenScreen 是一个基于 Electron 的开源桌面屏幕录制和视频编辑应用程序，提供类似于 Screen Studio 的功能，包括缩放、裁剪、注释和导出为 MP4/GIF 格式等高级功能。

### 核心特性

- 全屏或特定应用程序录制
- 手动缩放控制（6级深度）
- 自定义缩放持续时间和位置
- 裁剪视频录制区域
- 多种背景选择（壁纸、纯色、渐变、自定义图片）
- 运动模糊效果
- 添加注释（文本、箭头、图片）
- 剪辑视频片段
- 多种宽高比和分辨率导出选项

---

## 技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 桌面框架 | Electron | 39.2.7 |
| 前端框架 | React | 18.2.0 |
| 编程语言 | TypeScript | 5.2.2 |
| 构建工具 | Vite | 5.1.6 |
| 打包工具 | Electron Builder | 24.13.3 |
| 图形渲染 | PixiJS | 8.14.0 |
| 视频编码 | WebCodecs API | - |
| GIF 编码 | gif.js | 0.2.0 |
| 视频封装 | mediabunny | 1.25.1 |
| UI 组件 | Radix UI + Tailwind CSS | - |
| 动画库 | GSAP | 3.13.0 |
| 测试框架 | Vitest | 4.0.16 |

---

## 项目目录结构

```
openscreen/
├── electron/                    # Electron 主进程代码
│   ├── main.ts                 # 应用入口点，窗口生命周期
│   ├── windows.ts              # 窗口创建函数
│   ├── ipc/
│   │   └── handlers.ts         # IPC 通信处理器
│   └── preload.ts              # 上下文桥接（安全 IPC）
│
├── src/                        # 渲染进程（React UI）
│   ├── components/
│   │   ├── launch/             # 录制 UI 组件
│   │   │   ├── LaunchWindow.tsx        # HUD 录制控制界面
│   │   │   └── SourceSelector.tsx      # 屏幕源选择器
│   │   ├── ui/                 # shadcn/ui 基础组件
│   │   └── video-editor/       # 视频编辑器组件
│   │       ├── VideoEditor.tsx         # 主编辑器容器
│   │       ├── VideoPlayback.tsx       # PixiJS 视频播放器
│   │       ├── PlaybackControls.tsx    # 播放控制
│   │       ├── timeline/               # 时间轴组件
│   │       │   └── TimelineEditor.tsx
│   │       ├── videoPlayback/          # 视频渲染工具
│   │       │   ├── zoomTransform.ts    # 缩放变换计算
│   │       │   ├── layoutUtils.ts      # 布局工具
│   │       │   ├── focusUtils.ts       # 焦点工具
│   │       │   └── overlayUtils.ts     # 覆盖层工具
│   │       └── types.ts                # 类型定义
│   │
│   ├── hooks/
│   │   └── useScreenRecorder.ts        # 屏幕录制 Hook
│   │
│   ├── lib/
│   │   └── exporter/                   # 导出管道
│   │       ├── videoExporter.ts        # MP4 导出
│   │       ├── gifExporter.ts          # GIF 导出
│   │       ├── videoDecoder.ts         # 视频解码
│   │       ├── frameRenderer.ts        # 帧渲染
│   │       └── muxer.ts                # 视频封装
│   │
│   ├── utils/
│   │   ├── aspectRatioUtils.ts         # 宽高比工具
│   │   └── platformUtils.ts            # 平台检测
│   │
│   ├── App.tsx                 # 应用路由（根据 windowType 分发）
│   └── main.tsx                # React 入口点
│
├── public/                     # 静态资源
│   └── wallpapers/             # 18 张背景壁纸
│
├── icons/                      # 所有平台的应用图标
├── dist-electron/              # 编译后的 Electron 代码
└── dist/                       # Vite 构建输出
```

---

## 录制流程

### 1. 窗口管理

应用创建三种类型的窗口：

#### 1.1 HUD 覆盖窗口 (`windowType=hud-overlay`)
- **尺寸**: 500x100px
- **位置**: 屏幕底部中央
- **特点**: 无边框、透明、始终置顶
- **渲染**: `LaunchWindow.tsx` 组件
- **用途**: 显示录制控制和源选择器

#### 1.2 源选择器窗口 (`windowType=source-selector`)
- **尺寸**: 620x420px
- **位置**: 屏幕中央
- **用途**: 选择要录制的屏幕/窗口

#### 1.3 编辑器窗口 (`windowType=editor`)
- **尺寸**: 1200x800px（可调整）
- **状态**: 默认最大化
- **用途**: 主视频编辑界面

### 2. 录制过程

**核心 Hook**: `useScreenRecorder.ts`

#### 步骤详解：

```typescript
// 1. 源选择
const selectedSource = await window.electronAPI.getSelectedSource()

// 2. 获取媒体流
const mediaStream = await navigator.mediaDevices.getUserMedia({
  audio: false,
  video: {
    mandatory: {
      chromeMediaSource: "desktop",
      chromeMediaSourceId: selectedSource.id,
      maxWidth: 3840,
      maxHeight: 2160,
      maxFrameRate: 60,
      minFrameRate: 30,
    },
  },
})

// 3. 编解码器选择（优先级顺序）
// - video/webm;codecs=av1 (AV1)
// - video/webm;codecs=h264 (H.264)
// - video/webm;codecs=vp9 (VP9)
// - video/webm;codecs=vp8 (VP8)

// 4. 自适应比特率计算
if (pixels >= FOUR_K_PIXELS) {
  bitrate = 45_000_000 * highFrameRateBoost; // 4K@60: 45 Mbps
}

// 5. MediaRecorder 设置
const recorder = new MediaRecorder(stream, {
  mimeType,
  videoBitsPerSecond, // 根据分辨率 18-50 Mbps
})

// 6. 录制停止
// - 修复 WebM 时长
// - 转换为 ArrayBuffer
// - 通过 IPC 保存
// - 切换到编辑器窗口
```

### 3. 文件存储

- **存储位置**: `app.getPath('userData')/recordings/recording-{timestamp}.webm`
- **命名格式**: `recording-{时间戳}.webm`

---

## 编辑流程

### 1. 视频播放组件

**文件**: `VideoPlayback.tsx`

#### PixiJS 容器层次结构：

```
Stage (舞台)
└─ CameraContainer (相机容器 - 变换用于缩放)
   └─ VideoContainer (视频容器 - 遮罩和滤镜)
      ├─ VideoSprite (视频精灵 - 视频内容)
      └─ MaskGraphics (遮罩图形 - 圆角矩形遮罩)
```

#### 渲染管线：

1. **PixiJS 初始化**
2. **视频加载** - 从 video 元素创建纹理
3. **布局计算** - 裁剪、填充、圆角
4. **动画循环** (60 FPS)
   - 查找活动缩放区域
   - 计算目标缩放和焦点
   - 平滑插值变换
   - 应用模糊滤镜

### 2. 缩放系统

#### 数据结构：

```typescript
interface ZoomRegion {
  id: string
  startMs: number          // 开始时间（毫秒）
  endMs: number            // 结束时间（毫秒）
  depth: ZoomDepth         // 深度：1-6
  focus: {
    cx: number            // 0-1 水平中心（归一化）
    cy: number            // 0-1 垂直中心（归一化）
  }
}

const ZOOM_DEPTH_SCALES = {
  1: 1.25,   2: 1.5,    3: 1.8,
  4: 2.2,    5: 3.5,    6: 5.0
}
```

#### 变换逻辑：

```typescript
// 计算相机位置以保持焦点点居中
const focusStagePxX = focusX * stageSize.width
const focusStagePxY = focusY * stageSize.height
const stageCenterX = stageSize.width / 2
const cameraX = stageCenterX - focusStagePxX * zoomScale
cameraContainer.position.set(cameraX, cameraY)
```

#### 区域强度计算：

- 在区域边界处通过 500ms 缓入/缓出
- 在 0（无缩放）和 1（完全缩放）之间插值

#### 运动模糊：

```typescript
const motionIntensity = Math.max(
  Math.abs(nextScale - prevScale),
  Math.abs(nextFocusX - prevFocusX),
  Math.abs(nextFocusY - prevFocusY)
)
blurFilter.blur = motionBlurEnabled ? motionIntensity * 120 : 0
```

### 3. 时间轴编辑器

**文件**: `timeline/TimelineEditor.tsx`

#### 特性：

- 三个时间轴行：缩放、剪辑、注释
- 通过 `dnd-timeline` 库实现拖放调整
- 带有搜索的播放光标
- 自适应时间刻度（15 个候选，从 0.25s 到 1800s）

#### 键盘快捷键：

| 快捷键 | 功能 |
|--------|------|
| Z | 添加缩放区域 |
| T | 添加剪辑区域 |
| A | 添加注释 |
| Tab | 循环选择注释 |
| Ctrl+D | 删除选中项 |

### 4. 裁剪系统

```typescript
interface CropRegion {
  x: number      // 0-1（归一化）
  y: number      // 0-1
  width: number  // 0-1
  height: number // 0-1
}
```

### 5. 注释系统

#### 类型：`text`（文本）、`image`（图片）、`figure`（箭头）

```typescript
interface AnnotationRegion {
  id: string
  startMs: number
  endMs: number
  type: AnnotationType
  content: string
  position: { x: number; y: number }
  size: { width: number; height: number }
  style: {
    color: string
    backgroundColor: string
    fontSize: number
    fontFamily: string
    fontWeight: 'normal' | 'bold'
    // ...
  }
  zIndex: number  // 堆叠顺序
  figureData?: {
    arrowDirection: ArrowDirection
    color: string
    strokeWidth: number
  }
}
```

---

## 导出流程

### 导出管道架构

```
┌─────────────────────────────────────────────────────┐
│                   导出入口                           │
└─────────────────┬───────────────────────────────────┘
                  │
         ┌────────┴────────┐
         │                 │
    MP4 导出          GIF 导出
         │                 │
         └────────┬────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
VideoFileDecoder  │  FrameRenderer
    │             │             │
    └─────────────┼─────────────┘
                  │
         ┌────────┴────────┐
         │                 │
   VideoEncoder        GIF Encoder
         │                 │
         └────────┬────────┘
                  │
           VideoMuxer
                  │
            最终文件
```

### MP4 导出过程

**类**: `VideoExporter`

#### 导出步骤：

1. **初始化解码器**
   ```typescript
   this.decoder = new VideoFileDecoder()
   const videoInfo = await this.decoder.loadVideo(videoUrl)
   ```

2. **初始化渲染器**
   ```typescript
   this.renderer = new FrameRenderer(config)
   await this.renderer.initialize()
   ```

3. **初始化编码器** (WebCodecs)
   ```typescript
   this.encoder = new VideoEncoder({
     output: (chunk, meta) => {
       this.muxer.addVideoChunk(chunk, meta)
     },
     error: (error) => {
       this.cancelled = true
     }
   })

   await this.encoder.configure({
     codec: 'avc1.640033',
     width: exportWidth,
     height: exportHeight,
     bitrate: 30_000_000,  // 1080p: 30 Mbps
     framerate: 60,
     hardwareAcceleration: 'prefer-hardware',
   })
   ```

4. **处理帧**
   ```typescript
   while (frameIndex < totalFrames && !this.cancelled) {
     // 1. 查找源时间（考虑剪辑区域）
     const sourceTimeMs = this.mapEffectiveToSourceTime(effectiveTimeMs)

     // 2. 从视频元素创建 VideoFrame
     const videoFrame = new VideoFrame(videoElement, { timestamp })

     // 3. 使用效果渲染帧
     await this.renderer.renderFrame(videoFrame, sourceTimestamp)

     // 4. 从画布创建导出帧
     const exportFrame = new VideoFrame(canvas, { timestamp, duration })

     // 5. 编码帧
     this.encoder.encode(exportFrame, { keyFrame: frameIndex % 150 === 0 })

     // 6. 清理
     exportFrame.close()
     videoFrame.close()
   }
   ```

5. **完成**
   ```typescript
   await this.encoder.flush()
   await Promise.all(this.muxingPromises)
   const blob = await this.muxer.finalize()
   ```

### GIF 导出过程

**类**: `GifExporter`

#### GIF 编码器配置：

```typescript
this.gif = new GIF({
  workers: 4,
  quality: 10,
  width: this.config.width,
  height: this.config.height,
  workerScript: '/gif.worker.js',
  repeat: loop ? 0 : 1,  // 0 = 无限循环
  background: '#000000',
  dither: 'FloydSteinberg',
})
```

#### 处理帧：

```typescript
while (frameIndex < totalFrames && !this.cancelled) {
  // 渲染帧（与 MP4 相同）

  // 添加帧到 GIF
  this.gif.addFrame(canvas, {
    delay: Math.round(1000 / frameRate),  // 毫秒/帧
    copy: true
  })
}
```

#### 帧率选项：15、20、25、30 FPS

### 帧渲染

**类**: `FrameRenderer`

#### 渲染帧：

```typescript
async renderFrame(videoFrame: VideoFrame, timestamp: number) {
  // 1. 更新视频精灵纹理
  const texture = Texture.from(videoFrame)
  this.videoSprite.texture = texture

  // 2. 计算布局（裁剪、填充、圆角）
  this.updateLayout()

  // 3. 更新动画状态（缩放插值）
  this.updateAnimationState(timeMs)

  // 4. 应用缩放变换
  applyZoomTransform({
    cameraContainer,
    blurFilter,
    zoomScale: this.animationState.scale,
    focusX: this.animationState.focusX,
    focusY: this.animationState.focusY,
    motionIntensity,
  })

  // 5. 渲染 PixiJS 舞台到画布
  this.app.renderer.render(this.app.stage)

  // 6. 与背景和阴影合成
  this.compositeWithShadows()

  // 7. 在顶部渲染注释
  if (annotationRegions) {
    await renderAnnotations(
      this.compositeCtx,
      annotationRegions,
      this.config.width,
      this.config.height,
      timeMs,
      scaleFactor
    )
  }
}
```

### 合成过程

1. 绘制背景（带可选模糊）
2. 绘制阴影层（使用画布上的 CSS drop-shadow 滤镜）
3. 绘制 PixiJS 视频层
4. 在 2D 上下文上直接渲染注释

---

## 组件层次结构

```
App.tsx
│
├─ LaunchWindow.tsx (HUD 覆盖)
│  └─ useScreenRecorder Hook
│
├─ SourceSelector.tsx
│  └─ 源选择 UI
│
└─ VideoEditor.tsx
   │
   ├─ VideoPlayback.tsx
   │  ├─ PixiJS Application
   │  ├─ VideoContainer
   │  │  ├─ VideoSprite
   │  │  └─ MaskGraphics
   │  └─ AnnotationOverlay[]（活动注释）
   │
   ├─ PlaybackControls.tsx
   │
   ├─ TimelineEditor.tsx
   │  ├─ TimelineWrapper
   │  ├─ Timeline
   │  │  ├─ TimelineAxis
   │  │  ├─ PlaybackCursor
   │  │  ├─ Row（缩放）
   │  │  ├─ Row（剪辑）
   │  │  └─ Row（注释）
   │  └─ KeyframeMarkers
   │
   ├─ SettingsPanel.tsx
   │  ├─ 壁纸选择器
   │  ├─ 缩放深度控制
   │  ├─ 阴影/模糊控制
   │  ├─ 圆角/填充
   │  ├─ 裁剪控制
   │  ├─ 注释设置
   │  └─ 导出设置
   │
   └─ ExportDialog.tsx
```

---

## IPC 通信

### 架构：Electron IPC 与上下文桥接

### 主进程处理器

**文件**: `electron/ipc/handlers.ts`

#### 处理器类型：

1. **屏幕录制**
   - `get-sources`: 通过 `desktopCapturer` 获取可用屏幕/窗口
   - `select-source`: 存储选定的源
   - `get-selected-source`: 检索选定的源

2. **视频存储**
   - `store-recorded-video`: 将录制的视频保存到 userData 目录
   - `get-recorded-video-path`: 获取最新录制路径
   - `set-current-video-path`: 设置当前工作视频
   - `get-current-video-path`: 检索当前视频路径

3. **窗口管理**
   - `open-source-selector`: 打开源选择器窗口
   - `switch-to-editor`: 关闭 HUD，打开编辑器窗口

4. **导出**
   - `save-exported-video`: 打开保存对话框并写入文件

5. **系统**
   - `open-external-url`: 在浏览器中打开 URL
   - `get-asset-base-path`: 解析生产环境的资源路径
   - `get-platform`: 返回操作系统平台
   - `open-video-file-picker`: 打开文件选择对话框

### 渲染进程 API

**文件**: `electron/preload.ts`

#### 暴露的 API（通过 `contextBridge`）：

```typescript
window.electronAPI = {
  // 录制
  getSources(opts): Promise<Source[]>
  selectSource(source): Promise<Source>
  getSelectedSource(): Promise<Source>
  openSourceSelector(): Promise<void>
  setRecordingState(recording: boolean): Promise<void>

  // 视频
  storeRecordedVideo(data: ArrayBuffer, filename: string): Promise<Result>
  getRecordedVideoPath(): Promise<{ path: string }>
  setCurrentVideoPath(path: string): Promise<void>
  getCurrentVideoPath(): Promise<{ path: string }>
  clearCurrentVideoPath(): Promise<void>

  // 窗口
  switchToEditor(): Promise<void>
  hudOverlayHide(): void
  hudOverlayClose(): void

  // 导出
  saveExportedVideo(data: ArrayBuffer, filename: string): Promise<Result>
  openVideoFilePicker(): Promise<{ path: string }>

  // 系统
  openExternalUrl(url: string): Promise<void>
  getAssetBasePath(): Promise<string>
  getPlatform(): Promise<string>
}
```

### 通信流程示例

#### 录制到编辑器转换：

1. 用户在 HUD 中点击"停止录制"
2. `useScreenRecorder` 停止 MediaRecorder
3. 视频 blob 被修复并转换为 ArrayBuffer
4. IPC 调用：`storeRecordedVideo(arrayBuffer, fileName)`
5. 主进程写入到 `userData/recordings/`
6. IPC 调用：`switchToEditor()`
7. 主进程关闭 HUD 窗口，打开编辑器窗口
8. 编辑器窗口通过 `getCurrentVideoPath()` 加载视频路径

---

## 状态管理

### 方法：组件级状态与 React Hooks

### 主状态容器：`VideoEditor.tsx`（30+ 个状态变量）

#### 状态类别：

1. **视频播放**
   ```typescript
   const [currentTime, setCurrentTime] = useState(0)
   const [duration, setDuration] = useState(0)
   const [isPlaying, setIsPlaying] = useState(false)
   ```

2. **编辑区域**（集合）
   ```typescript
   const [zoomRegions, setZoomRegions] = useState<ZoomRegion[]>([])
   const [trimRegions, setTrimRegions] = useState<TrimRegion[]>([])
   const [annotationRegions, setAnnotationRegions] = useState<AnnotationRegion[]>([])
   ```

3. **选择状态**
   ```typescript
   const [selectedZoomId, setSelectedZoomId] = useState<string | null>(null)
   const [selectedTrimId, setSelectedTrimId] = useState<string | null>(null)
   const [selectedAnnotationId, setSelectedAnnotationId] = useState<string | null>(null)
   ```

4. **视觉设置**
   ```typescript
   const [wallpaper, setWallpaper] = useState<string>(defaultPath)
   const [shadowIntensity, setShadowIntensity] = useState(0)
   const [showBlur, setShowBlur] = useState(false)
   const [motionBlurEnabled, setMotionBlurEnabled] = useState(true)
   const [borderRadius, setBorderRadius] = useState(0)
   const [padding, setPadding] = useState(50)
   const [cropRegion, setCropRegion] = useState<CropRegion>(DEFAULT_CROP_REGION)
   ```

5. **导出设置**
   ```typescript
   const [exportFormat, setExportFormat] = useState<ExportFormat>('mp4')
   const [exportQuality, setExportQuality] = useState<ExportQuality>('good')
   const [gifFrameRate, setGifFrameRate] = useState<GifFrameRate>(15)
   const [gifLoop, setGifLoop] = useState(true)
   const [gifSizePreset, setGifSizePreset] = useState<GifSizePreset>('medium')
   ```

6. **导出进度**
   ```typescript
   const [isExporting, setIsExporting] = useState(false)
   const [exportProgress, setExportProgress] = useState<ExportProgress | null>(null)
   const [exportError, setExportError] = useState<string | null>(null)
   const [showExportDialog, setShowExportDialog] = useState(false)
   ```

#### 状态更新：所有更新通过回调 props

- 示例：`onZoomFocusChange={(id, focus) => setZoomRegions(...)}`
- 整个应用使用 props drilling（无全局状态管理）

---

## 核心数据结构

### 类型定义

**文件**: `src/components/video-editor/types.ts`

#### 缩放区域

```typescript
type ZoomDepth = 1 | 2 | 3 | 4 | 5 | 6

interface ZoomFocus {
  cx: number  // 0-1，归一化水平中心
  cy: number  // 0-1，归一化垂直中心
}

interface ZoomRegion {
  id: string
  startMs: number
  endMs: number
  depth: ZoomDepth
  focus: ZoomFocus
}
```

#### 剪辑区域

```typescript
interface TrimRegion {
  id: string
  startMs: number
  endMs: number
}
```

#### 注释区域

```typescript
type AnnotationType = 'text' | 'image' | 'figure'

interface AnnotationPosition {
  x: number
  y: number
}

interface AnnotationSize {
  width: number
  height: number
}

interface AnnotationTextStyle {
  color: string
  backgroundColor: string
  fontSize: number
  fontFamily: string
  fontWeight: 'normal' | 'bold'
  fontStyle: 'normal' | 'italic'
  textDecoration: 'none' | 'underline'
  textAlign: 'left' | 'center' | 'right'
}

interface AnnotationRegion {
  id: string
  startMs: number
  endMs: number
  type: AnnotationType
  content: string
  textContent?: string
  imageContent?: string
  position: AnnotationPosition
  size: AnnotationSize
  style: AnnotationTextStyle
  zIndex: number
}
```

#### 裁剪区域

```typescript
interface CropRegion {
  x: number
  y: number
  width: number
  height: number
}
```

### 导出类型

**文件**: `src/lib/exporter/types.ts`

```typescript
interface ExportProgress {
  currentFrame: number
  totalFrames: number
  percentage: number
  estimatedTimeRemaining: number
  phase?: 'extracting' | 'finalizing'
  renderProgress?: number
}

type ExportQuality = 'medium' | 'good' | 'source'
type ExportFormat = 'mp4' | 'gif'

interface GifExportConfig {
  frameRate: 15 | 20 | 25 | 30
  loop: boolean
  sizePreset: 'medium' | 'large' | 'original'
  width: number
  height: number
}
```

---

## 完整工作流

### 录制 → 编辑 → 导出

```
┌─────────────────────────────────────────────────────────────┐
│                        录制阶段                              │
├─────────────────────────────────────────────────────────────┤
│ 1. 用户选择屏幕/窗口源                                        │
│ 2. useScreenRecorder 通过 MediaRecorder API 捕获            │
│ 3. 视频作为 WebM 存储在 userData 目录                        │
│ 4. 窗口切换到编辑器                                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                        编辑阶段                              │
├─────────────────────────────────────────────────────────────┤
│ 1. 视频在 VideoPlayback 组件中加载                           │
│ 2. PixiJS 使用 GPU 加速渲染视频                             │
│ 3. 用户通过时间轴添加缩放/剪辑/注释区域                       │
│ 4. 调整视觉设置（壁纸、阴影、模糊等）                         │
│ 5. 带有平滑动画的实时预览                                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                        导出阶段                              │
├─────────────────────────────────────────────────────────────┤
│ 1. 用户点击导出（MP4 或 GIF）                                │
│ 2. FrameRenderer 处理每帧并应用效果                          │
│ 3. WebCodecs (MP4) 或 gif.js (GIF) 编码帧                   │
│ 4. Mediabunny 将视频封装到 MP4 容器                         │
│ 5. 通过原生保存对话框保存文件                                │
│ 6. Toast 通知确认成功                                        │
└─────────────────────────────────────────────────────────────┘
```

### 关键技术亮点

| 特性 | 实现 |
|------|------|
| GPU 加速视频渲染 | PixiJS |
| 硬件视频编码 | WebCodecs API |
| 平滑缩放插值 | 运动模糊 |
| 剪辑感知导出时间线 | 时间映射 |
| 实时预览 | 精确导出渲染 |
| 高效内存管理 | 视频帧池化 |
| 并行编码 | 队列管理 |

### 文件位置

| 类型 | 路径 |
|------|------|
| 录制文件 | `app.getPath('userData')/recordings/` |
| 资源文件 | `process.resourcesPath/assets/` (生产) 或 `public/assets/` (开发) |
| 导出文件 | 用户通过原生保存对话框选择的位置 |

---

## 总结

OpenScreen 是一个功能完整的屏幕录制和视频编辑应用程序，其架构设计充分考虑了性能和用户体验：

1. **模块化设计**: 清晰的进程分离和组件层次结构
2. **GPU 加速**: 使用 PixiJS 实现流畅的视频渲染
3. **硬件编码**: 利用 WebCodecs API 实现高效导出
4. **用户友好**: 直观的时间轴编辑和实时预览
5. **跨平台**: 支持 Windows、macOS 和 Linux
