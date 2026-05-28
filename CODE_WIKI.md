# Gallery2 Code Wiki

## 项目概述

**项目名称**: Gallery2  
**项目类型**: Android 原生图库应用  
**版本**: 1.1.40030  
**包名**: com.android.gallery3d  
**最低SDK版本**: 14 (Android 4.0)  
**目标SDK版本**: 28 (Android 9.0)  
**许可证**: Apache-2.0

> **注意**: 此应用已不再积极维护，源代码仅作为参考提供。

---

## 目录

1. [项目整体架构](#项目整体架构)
2. [主要模块职责](#主要模块职责)
3. [关键类与函数说明](#关键类与函数说明)
4. [依赖关系](#依赖关系)
5. [项目运行方式](#项目运行方式)
6. [代码统计](#代码统计)

---

## 项目整体架构

Gallery2 是一个功能完整的Android图库应用，采用模块化架构设计，主要包含以下核心层次：

```
┌─────────────────────────────────────────────────────────┐
│                      UI层 (UI Layer                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │相册   │  │ 相机   │  │照片页  │  │滤镜编辑 │ │
│  │浏览   │  │模块    │  │查看   │  │器      │ │
│  └─────┘  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   应用层 (App Layer)                   │
│  GalleryAppImpl / StateManager / ActivityState            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                  数据层 (Data Layer)                      │
│  DataManager / MediaSet / MediaItem / 各类Source    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│               渲染层 (Rendering Layer)                    │
│  GLCanvas / Texture / OpenGL ES 渲染                      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                JNI 层 (Native Layer)                       │
│  jni_jpegstream / jni_filtershow_filters              │
└─────────────────────────────────────────────────────────┘
```

---

## 主要模块职责

### 1. app 模块 - 应用主模块

**位置**: `src/com/android/gallery3d/app/`

**主要类职责**:

- **GalleryAppImpl**: 应用入口，管理全局资源和服务
  - 数据管理：管理应用生命周期
  - 提供 DataManager、ImageCacheService、ThreadPool 等核心服务

- **GalleryActivity**: 主界面 Activity
  - 相册浏览入口
  - 处理照片/视频选择和查看

- **StateManager**: 页面状态管理器
  - 管理 ActivityState 栈
  - 处理页面切换和回退

- **AlbumPage / PhotoPage**: 核心页面
  - 相册页面和照片查看页面

- **MovieActivity**: 视频播放器
  - 支持多种视频格式播放

**关键文件**:
  - [GalleryAppImpl.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryAppImpl.java)
  - [GalleryActivity.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryActivity.java)

---

### 2. data 模块 - 数据管理模块

**位置**: `src/com/android/gallery3d/data/`

**职责**:
- 媒体数据管理
- 相册、相册集、媒体项抽象
- 数据源管理（本地、MTP等

**核心类**:

- **DataManager**: 数据管理中枢
  - 管理所有 MediaSource
  - 路径解析和匹配

- **MediaSet**: 媒体集合基类
  - LocalAlbumSet: 本地相册集
  - LocalAlbum: 本地相册
  - ClusterAlbum: 聚类相册

- **MediaItem**: 媒体项基类
  - LocalImage: 本地图片
  - LocalVideo: 本地视频
  - UriImage: URI图片

- **MediaSource**: 数据源接口
  - LocalSource: 本地媒体源
  - UriSource: URI媒体源
  - MtpSource: MTP设备媒体源

---

### 3. filtershow 模块 - 照片编辑器

**位置**: `src/com/android/gallery3d/filtershow/`

**职责**: 提供强大的照片编辑功能

**子模块**:

| 子模块 | 功能 |
|--------|------|
| **filters/** | 各种滤镜实现 |
| **editors/** | 各类编辑器实现 |
| **pipeline/** | 图像处理流水线 |
| **crop/** | 裁剪功能 |
| **history/** | 历史记录管理 |
| **state/** | 状态管理 |

**主要功能**:
- 基础调整：亮度、对比度、饱和度、曝光
- 滤镜效果：复古、即时、漂白等
- 几何变换：裁剪、旋转、镜像
- 高级编辑：曲线、红眼消除、小星球效果

**关键类**:
- **FilterShowActivity**: 编辑器主界面
- **FiltersManager**: 滤镜管理器
- **ImageFilter**: 滤镜基类
- **ProcessingService**: 后台处理服务

---

### 4. ui 模块 - 用户界面组件

**位置**: `src/com/android/gallery3d/ui/`

**职责**: 提供自定义UI组件

**核心组件**:

- **GLView / GLRootView**: OpenGL 视图体系
- **SlotView**: 缩略图网格视图
- **PhotoView**: 照片查看视图
- **AlbumSlidingWindow**: 相册滑动窗口
- **SelectionManager**: 选择管理器

---

### 5. glrenderer 模块 - OpenGL 渲染

**位置**: `src/com/android/gallery3d/glrenderer/`

**职责**: 基于 OpenGL ES 渲染引擎

**核心类**:
- **GLCanvas**: 画布抽象
- **GLES11Canvas / GLES20Canvas**: OpenGL 具体实现
- **Texture**: 纹理系统
- **BitmapTexture**: 位图纹理

---

### 6. gallerycommon 模块 - 公共库

**位置**: `gallerycommon/src/com/android/gallery3d/`

**职责**: 提供公共工具和通用组件

**子模块**:

| 模块 | 功能 |
|-----|------|
| **exif/** | EXIF 数据处理 |
| **jpegstream/** | JPEG 流处理 |
| **common/** | 通用工具类 |
| **util/** | 线程池等工具 |

---

### 7. ingest 模块 - 媒体导入

**位置**: `src/com/android/gallery3d/ingest/`

**职责**: 从 MTP 设备导入媒体

**功能**:
- 连接 MTP 设备
- 显示设备上的媒体
- 批量导入照片和视频

---

### 8. gadget 模块 - 桌面小部件

**位置**: `src/com/android/gallery3d/gadget/`

**职责**: 提供相册桌面小部件功能

---

### 9. JNI 原生模块

**位置**: `jni/` 和 `jni_jpegstream/`

**职责**: 提供高性能图像处理

**子模块**:

| 模块 | 功能 |
|-----|------|
| **jni/filters/** | C 实现的滤镜 |
| **jni_jpegstream/** | JPEG 流处理 |

**关键文件**:
  - [bwfilter.c](file:///d:/code/gallery2/Gallery2/jni/filters/bwfilter.c)
  - [jpegstream.cpp](file:///d:/code/gallery2/Gallery2/jni_jpegstream/src/jpegstream.cpp)

---

### 10. photos 模块 - 新照片查看模块

**位置**: `src/com/android/photos/`

**职责**: 提供现代化的照片查看体验

---

## 关键类与函数说明

### 核心类详解

#### 1. GalleryAppImpl

**文件**: [GalleryAppImpl.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryAppImpl.java)

**主要方法**:

| 方法 | 说明 |
|------|------|
| `onCreate()` | 应用初始化 |
| `getDataManager()` | 获取数据管理器 |
| `getImageCacheService()` | 获取图片缓存服务 |
| `getThreadPool()` | 获取线程池 |
| `getDownloadCache()` | 获取下载缓存 |

---

#### 2. DataManager

**职责**: 媒体数据管理中心

**主要方法**:
- `initializeSourceMap()`: 初始化数据源
- `getMediaObject()`: 获取媒体对象
- `registerSource()`: 注册数据源

---

#### 3. MediaSet

**职责**: 媒体集合基类

**主要方法**:
- `getMediaItemCount()`: 获取媒体项数量
- `getMediaItem()`: 获取媒体项
- `getSubMediaSetCount()`: 获取子相册数量

---

#### 4. MediaItem

**职责**: 媒体项基类

**主要方法**:
- `requestImage()`: 请求图片
- `getMimeType()`: 获取 MIME 类型
- `getFilePath()`: 获取文件路径

---

#### 5. GLCanvas

**职责**: OpenGL 画布抽象

**主要方法**:
- `drawTexture()`: 绘制纹理
- `drawRect()`: 绘制矩形
- `save()` / `restore()`: 保存/恢复状态

---

#### 6. FilterShowActivity

**文件**: [FilterShowActivity.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/filtershow/FilterShowActivity.java)

**职责**: 照片编辑器主界面

**功能**:
- 加载和保存图片
- 应用滤镜
- 历史记录管理
- 参数调整

---

#### 7. GalleryActivity

**文件**: [GalleryActivity.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryActivity.java)

**职责**: 主界面 Activity

**Intent 过滤器**:
- `android.intent.action.MAIN (LAUNCHER)
- `android.intent.action.GET_CONTENT` (获取内容)
- `android.intent.action.VIEW` (查看图片/视频)
- `android.intent.action.PICK` (选择媒体)

---

## 依赖关系

### 内部模块依赖

```
Gallery2
├── androidx.fragment_fragment
├── androidx.legacy_legacy-support-core-ui
├── androidx.core_core
├── androidx.legacy_legacy-support-v13
├── com.android.gallery3d.common2 (公共库
├── xmp_toolkit (XMP 处理
└── mp4parser (MP4 解析)
```

### JNI 库依赖

```
jni_libs:
├── libjni_eglfence
├── libjni_filtershow_filters
└── libjni_jpegstream
```

### 可选库依赖

```
optional_uses_libs:
├── com.google.android.media.effects
└── org.apache.http.legacy
```

### AndroidManifest 权限

**权限列表**:

| 权限 | 用途 |
|-----|------|
| ACCESS_COARSE_LOCATION | 粗略位置 |
| ACCESS_FINE_LOCATION | 精确位置 |
| CAMERA | 相机 |
| INTERNET | 网络 |
| READ_EXTERNAL_STORAGE | 读取外部存储 |
| WRITE_EXTERNAL_STORAGE | 写入外部存储 |
| RECORD_AUDIO | 录音 |
| WAKE_LOCK | 唤醒锁 |
| NFC | NFC |
| SET_WALLPAPER | 设置壁纸 |

---

## 项目运行方式

### 构建系统

项目使用 Android Soong 构建系统（Android.bp）

### 主要组件

**Activity 列表:

| Activity | 功能 |
|----------|------|
| GalleryActivity | 主界面 |
| FilterShowActivity | 照片编辑器 |
| CropActivity | 裁剪 |
| MovieActivity | 视频播放器 |
| IngestActivity | MTP导入 |
| Wallpaper | 壁纸设置 |
| TrimVideo | 视频剪辑 |

**Service 列表**:

| Service | 功能 |
|---------|------|
| IngestService | MTP导入服务 |
| ProcessingService | 滤镜处理服务 |
| WidgetService | 小部件服务 |
| BatchService | 批量处理服务 |
| MediaSaveService | 媒体保存服务 |

**Provider 列表**:

| Provider | 功能 |
|----------|------|
| GalleryProvider | 图库内容提供器 |
| PhotoProvider | 照片内容提供器 |
| SharedImageProvider | 共享图片提供器 |
| FileProvider | 文件提供器 |

---

## 代码统计

### 总体统计

| 语言 | 文件数 | 代码行数（去除空行/注释） |
|------|--------|--------------------------|
| Java | 513 | 71,331 |
| C | 17 | 932 |
| C++ | 9 | 969 |
| C Header | 13 | 281 |
| **总计** | **552** | **73,513** |

### 按模块代码分布

主要代码主要集中在以下模块：
- app: 应用主模块
- filtershow: 照片编辑
- data: 数据管理
- ui: 用户界面
- glrenderer: OpenGL渲染

---

## 项目配置

### Android.bp 配置要点:

```bp
android_app {
    name: "Gallery2",
    sdk_version: "current",
    target_sdk_version: "28",
    product_specific: true,
    largeHeap: true,
    hardwareAccelerated: true,
}
```

### 目录结构

```
Gallery2/
├── src/                    # 主源代码
├── src_pd/                # 产品特定代码
├── gallerycommon/         # 公共库
├── jni/                   # JNI 滤镜
├── jni_jpegstream/       # JPEG流处理
├── res/                   # 资源文件
├── AndroidManifest.xml    # 应用清单
├── Android.bp            # Soong 构建文件
└── proguard.flags        # ProGuard 配置
```

---

## 关键功能流程

### 1. 相册浏览流程

```
GalleryActivity
    ↓
StateManager → AlbumPage
    ↓
DataManager → LocalAlbumSet / LocalAlbum
    ↓
AlbumSlotRenderer → 渲染缩略图
```

### 2. 照片编辑流程

```
FilterShowActivity
    ↓
FiltersManager → 加载滤镜
    ↓
ProcessingService → 后台处理
    ↓
Pipeline → 应用滤镜
    ↓
ImageShow → 显示结果
```

### 3. 数据加载流程

```
MediaItem.requestImage()
    ↓
ImageCacheService
    ↓
ThreadPool → 后台解码
    ↓
BitmapTexture → GLView.render()
```

---

## 开发注意事项

1. **已废弃声明: 此项目已不再积极维护，仅作为参考
2. **目标平台**: Android 9.0 (API 28)
3. **OpenGL ES 渲染**: 大量使用 OpenGL ES 进行渲染
4. **JNI 性能优化**: 滤镜和JPEG处理使用 C/C++ 优化
5. **线程模型**: 大量使用 ThreadPool 进行后台处理
6. **状态管理**: 使用 StateManager 管理页面状态
7. **数据抽象**: MediaSet/MediaItem 抽象体系

---

## 总结

Gallery2 是一个功能完整、架构清晰的 Android 原生图库应用。虽然已不再维护，但其代码质量高、架构设计优秀，是学习 Android 多媒体应用开发的宝贵参考资料。特别是其 OpenGL ES 渲染、照片编辑、媒体管理等模块都值得深入学习和借鉴。
