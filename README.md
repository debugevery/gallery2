# Gallery2 源码分析项目

## 项目简介

本项目是对 Android 系统原生图库应用 **Gallery2** 的深度技术分析文档集合。Gallery2 是 Android 4.0-9.0 系统内置的图库应用，采用 OpenGL ES 渲染引擎，代码质量高、架构设计优秀，是学习 Android 多媒体应用开发的重要参考资料。

## 核心分析文档

| 文档 | 说明 |
|-----|------|
| [CODE_WIKI.md](CODE_WIKI.md) | Gallery2 代码维基百科，系统性介绍项目架构、核心类、模块职责 |
| [GALLERY2_DEEP_DIVE.md](GALLERY2_DEEP_DIVE.md) | 深度分析文档，剖析应用初始化、页面框架、交互流程 |
| [宫格页加载逻辑深度分析.md](宫格页加载逻辑深度分析.md) | 宫格页完整加载流程，包含线程模型、数据同步、渲染机制、缓存体系 |

## 技术栈

- **平台**: Android (API 14-28)
- **语言**: Java + JNI (C/C++)
- **渲染引擎**: OpenGL ES (1.1/2.0)
- **构建系统**: Android Soong
- **代码规模**: 71,331 行 Java + 2,182 行 C/C++

## 核心架构

```
Gallery2
├── app/              # 应用层 (Activity、状态管理)
├── data/             # 数据层 (MediaSet、MediaItem)
├── ui/               # UI组件 (GLView、SlotView)
├── glrenderer/       # OpenGL渲染 (GLCanvas、Texture)
├── filtershow/       # 照片编辑器 (滤镜、管线)
├── gallerycommon/    # 公共库 (线程池、EXIF、JPEG)
└── jni/              # 原生代码 (性能优化滤镜)
```

## 主要特性

- **OpenGL 渲染**: 自定义基于 OpenGL ES 的视图系统，性能优异
- **状态管理**: 类似 Fragment 的栈式状态管理机制
- **数据抽象**: MediaSet/MediaItem 抽象，支持多数据源
- **渐进式渲染**: TileImageView 按需加载瓦片，处理大照片
- **缓存体系**: LRU 内存缓存 + 磁盘缓存 + Bitmap 复用池

## 核心模块分析

### 1. 宫格页加载机制

分析文档: [宫格页加载逻辑深度分析.md](宫格页加载逻辑深度分析.md)

- **线程模型**: ThreadPool (4核心/8最大) + ReloadTask 独立线程
- **数据同步**: ContentObserver 监听 MediaStore 变化
- **滑动窗口**: AlbumSlidingWindow 只缓存可见区域 (96槽)
- **渲染流程**: SlotView → AlbumSlotRenderer → OpenGL

### 2. 页面框架

分析文档: [GALLERY2_DEEP_DIVE.md](GALLERY2_DEEP_DIVE.md) 第三章

- **StateManager**: 栈式状态管理
- **ActivityState**: 页面状态基类
- **GLView**: 自定义 OpenGL 视图体系

### 3. 照片查看

分析文档: [GALLERY2_DEEP_DIVE.md](GALLERY2_DEEP_DIVE.md) 第五章

- **PhotoView**: 支持缩放、拖拽、滑动切换
- **TileImageView**: 高清瓦片渲染
- **GestureRecognizer**: 手势识别

## 快速导航

- [完整项目架构图](CODE_WIKI.md#项目整体架构)
- [核心类索引](CODE_WIKI.md#关键类与函数说明)
- [线程模型详解](宫格页加载逻辑深度分析.md#1-线程模型分析)
- [LRU缓存体系](宫格页加载逻辑深度分析.md#4-lru-缓存体系)
- [完整数据流图](宫格页加载逻辑深度分析.md#6-完整数据流图)

## 学习价值

虽然 Gallery2 已不再积极维护，但其架构设计思想仍具有重要参考价值：

1. **自定义 UI 框架**: 基于 OpenGL ES 的完整视图系统
2. **异步加载设计**: 多线程协作 + Handler 通信
3. **缓存策略**: 多级缓存 + Bitmap 复用
4. **性能优化**: 瓦片渲染 + 滑动窗口 + 按需加载

## 项目信息

- **版本**: 1.1.40030
- **包名**: com.android.gallery3d
- **许可证**: Apache-2.0
- **最低SDK**: 14 (Android 4.0)
- **目标SDK**: 28 (Android 9.0)
