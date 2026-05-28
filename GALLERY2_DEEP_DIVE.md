# Gallery2 深度分析文档

## 目录
1. [项目概述](#项目概述)
2. [应用初始化流程](#应用初始化流程)
3. [外层交互接口分析](#外层交互接口分析)
4. [框架设计逻辑](#框架设计逻辑)
5. [宫格页呈现分析](#宫格页呈现分析)
6. [大图页绘制分析](#大图页绘制分析)
7. [完整交互流程图](#完整交互流程图)
8. [核心模块汇总](#核心模块汇总)

---

## 项目概述

Gallery2 是 Android 系统原生相册应用，采用 OpenGL ES 渲染引擎，包含以下核心特性：
- 相册浏览、照片查看、视频播放
- 照片编辑（滤镜、裁剪、旋转等）
- 媒体导入（从 MTP 设备）
- 桌面小部件
- 全景照片支持

**技术栈**：
- 语言：Java + JNI (C/C++)
- 渲染：OpenGL ES (1.1/2.0)
- 构建：Android Soong 构建系统
- 最低 SDK：14 (Android 4.0)
- 目标 SDK：28 (Android 9.0)

---

## 应用初始化流程

### 1.1 应用启动入口

```
AndroidManifest.xml
    ↓
GalleryAppImpl (Application)
    ↓
GalleryActivity
    ↓
AbstractGalleryActivity
    ↓
StateManager
    ↓
AlbumSetPage (默认首页)
```

### 1.2 GalleryAppImpl 初始化

**文件位置**：[GalleryAppImpl.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryAppImpl.java)

```java
public class GalleryAppImpl extends Application implements GalleryApp {
    private ImageCacheService mImageCacheService;
    private DataManager mDataManager;
    private ThreadPool mThreadPool;
    private DownloadCache mDownloadCache;

    @Override
    public void onCreate() {
        super.onCreate();
        // 1. 初始化 AsyncTask（必须在 UI 线程）
        initializeAsyncTask();
        // 2. 初始化 GalleryUtils
        GalleryUtils.initialize(this);
        // 3. 初始化 WidgetUtils
        WidgetUtils.initialize(this);
        // 4. 初始化 PicasaSource
        PicasaSource.initialize(this);
        // 5. 初始化 UsageStatistics
        UsageStatistics.initialize(this);
    }
}
```

**核心服务**：
- `DataManager`：媒体数据管理中枢
- `ImageCacheService`：图片缓存服务
- `ThreadPool`：线程池，处理后台任务
- `DownloadCache`：下载缓存

### 1.3 GalleryActivity 启动流程

**文件位置**：[GalleryActivity.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryActivity.java)

```
onCreate()
    ↓
setContentView(R.layout.main)  // 加载布局
    ↓
if (savedInstanceState != null)
    getStateManager().restoreFromState()  // 恢复状态
else
    initializeByIntent()  // 根据 Intent 初始化
```

### 1.4 Intent 路由逻辑

`initializeByIntent()` 方法根据不同的 Intent Action 路由到不同页面：

| Intent Action | 跳转目标 | 用途 |
|--------------|---------|------|
| `ACTION_GET_CONTENT` | AlbumSetPage | 选择媒体 |
| `ACTION_PICK` | AlbumSetPage | 选择媒体（向后兼容） |
| `ACTION_VIEW` | SinglePhotoPage/MovieActivity | 查看单个媒体 |
| `ACTION_REVIEW` | SinglePhotoPage/MovieActivity | 相机刚拍摄的媒体 |
| (默认) | AlbumSetPage | 正常启动 |

---

## 外层交互接口分析

### 2.1 GalleryActivity Intent Filters

**AndroidManifest.xml** 中的主要 Intent 过滤器：

```xml
<!-- 主入口 -->
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.LAUNCHER" />
    <category android:name="android.intent.category.APP_GALLERY" />
</intent-filter>

<!-- 获取内容 -->
<intent-filter>
    <action android:name="android.intent.action.GET_CONTENT" />
    <category android:name="android.intent.category.OPENABLE" />
    <data android:mimeType="image/*" />
    <data android:mimeType="video/*" />
</intent-filter>

<!-- 查看媒体 -->
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <action android:name="com.android.camera.action.REVIEW" />
    <data android:scheme="content" />
    <data android:scheme="file" />
    <data android:mimeType="image/*" />
    <data android:mimeType="video/*" />
</intent-filter>
```

### 2.2 对外暴露的 Activity 列表

| Activity | 用途 |
|----------|------|
| `GalleryActivity` | 主界面，相册浏览 |
| `FilterShowActivity` | 照片编辑器 |
| `CropActivity` | 裁剪工具 |
| `MovieActivity` | 视频播放器 |
| `IngestActivity` | MTP 媒体导入 |
| `Wallpaper` | 壁纸设置 |
| `TrimVideo` | 视频剪辑 |

---

## 框架设计逻辑

### 3.1 StateManager - 状态管理器

**文件位置**：[StateManager.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/StateManager.java)

**核心设计**：使用 Stack 管理 ActivityState，实现类似 Fragment 的栈结构。

```java
public class StateManager {
    private Stack<StateEntry> mStack = new Stack<StateEntry>();
    
    // 关键方法
    public void startState(Class<? extends ActivityState> klass, Bundle data)
    public void startStateForResult(Class<? extends ActivityState> klass, 
                                   int requestCode, Bundle data)
    public void finishState(ActivityState state)
    public void switchState(ActivityState oldState, 
                          Class<? extends ActivityState> klass, 
                          Bundle data)
    public void saveState(Bundle outState)
    public void restoreFromState(Bundle inState)
}
```

**状态栈操作流程**：

```
启动新状态 (startState):
  1. 创建 ActivityState 实例
  2. 如果栈非空，暂停顶部状态
  3. 调用 initialize() 和 onCreate()
  4. 压入栈
  5. 如果已 resume，调用 resume()

结束状态 (finishState):
  1. 检查是否是最后一个状态（是则 finish Activity）
  2. 弹出栈
  3. 暂停并销毁当前状态
  4. 恢复上一个状态
```

### 3.2 ActivityState - 页面状态基类

**文件位置**：[ActivityState.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/ActivityState.java)

```java
abstract public class ActivityState {
    protected AbstractGalleryActivity mActivity;
    protected Bundle mData;
    protected int mFlags;
    
    // 生命周期方法
    protected void onCreate(Bundle data, Bundle storedState) { }
    protected void onResume() { }
    protected void onPause() { }
    protected void onDestroy() { }
    protected void onBackPressed() {
        mActivity.getStateManager().finishState(this);
    }
    
    // UI 相关
    protected void setContentPane(GLView content)
    protected int getBackgroundColorId()
}
```

**核心子类**：
- `AlbumSetPage`：相册集页面（显示所有相册）
- `AlbumPage`：相册页面（宫格显示照片）
- `PhotoPage` / `SinglePhotoPage`：照片查看页面
- `FilmstripPage`：胶片浏览页面
- `SlideshowPage`：幻灯片页面

### 3.3 AbstractGalleryActivity - Activity 基类

**文件位置**：[AbstractGalleryActivity.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AbstractGalleryActivity.java)

```java
public class AbstractGalleryActivity extends Activity implements GalleryContext {
    private GLRootView mGLRootView;        // OpenGL 根视图
    private StateManager mStateManager;    // 状态管理器
    private GalleryActionBar mActionBar;   // 自定义 ActionBar
    private OrientationManager mOrientationManager;  // 方向管理
    
    @Override
    public void setContentView(int resId) {
        super.setContentView(resId);
        mGLRootView = (GLRootView) findViewById(R.id.gl_root_view);
    }
    
    // 生命周期委托给 StateManager
    @Override
    protected void onResume() {
        super.onResume();
        mGLRootView.lockRenderThread();
        try {
            getStateManager().resume();
            getDataManager().resume();
        } finally {
            mGLRootView.unlockRenderThread();
        }
        mGLRootView.onResume();
    }
}
```

### 3.4 GLView - 自定义视图体系

Gallery2 没有使用 Android 原生 View 体系，而是基于 OpenGL ES 实现了一套完整的自定义视图体系。

```
GLView (基类)
    ├── GLRootView (根视图，关联 SurfaceView)
    ├── SlotView (宫格视图)
    ├── PhotoView (照片视图)
    ├── TileImageView (瓦片视图)
    └── ...
```

**GLView 核心方法**：
- `render(GLCanvas canvas)`：渲染
- `onLayout(...)`：布局
- `onTouch(...)`：触摸事件

### 3.5 GLCanvas - OpenGL 画布抽象

**文件位置**：[GLCanvas.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/GLCanvas.java)

```java
public interface GLCanvas {
    // 基本绘制
    void drawTexture(BasicTexture texture, int x, int y, int width, int height);
    void drawRect(float x1, float y1, float x2, float y2, GLPaint paint);
    void fillRect(float x, float y, float width, float height, int color);
    
    // 矩阵变换
    void translate(float x, float y, float z);
    void scale(float sx, float sy, float sz);
    void rotate(float angle, float x, float y, float z);
    void multiplyMatrix(float[] mMatrix, int offset);
    
    // 状态管理
    void save();
    void save(int saveFlags);
    void restore();
    
    // Alpha 控制
    void setAlpha(float alpha);
    void multiplyAlpha(float alpha);
}
```

**实现类**：
- `GLES11Canvas`：OpenGL ES 1.1 实现
- `GLES20Canvas`：OpenGL ES 2.0 实现

---

## 宫格页呈现分析

### 4.1 AlbumPage 整体架构

**文件位置**：[AlbumPage.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumPage.java)

```
AlbumPage (ActivityState)
    ├── mRootPane (GLView)
    │   └── mSlotView (SlotView)
    │
    ├── mAlbumView (AlbumSlotRenderer)
    │
    ├── mAlbumDataAdapter (AlbumDataLoader)
    │
    ├── mMediaSet (MediaSet)
    │
    └── mSelectionManager (SelectionManager)
```

### 4.2 AlbumPage 初始化流程

```
onCreate()
    ├── initializeViews()
    │   ├── 创建 SlotView
    │   ├── 创建 AlbumSlotRenderer
    │   ├── 设置 SlotRenderer
    │   └── 设置 Listener
    │
    └── initializeData()
        ├── 获取 MediaSet 路径
        ├── 创建 AlbumDataLoader
        └── 设置 Model

onResume()
    ├── setContentPane(mRootPane)
    ├── mAlbumDataAdapter.resume()
    ├── mAlbumView.resume()
    └── mMediaSet.requestSync()  // 同步媒体数据
```

### 4.3 SlotView - 宫格容器

**文件位置**：[SlotView.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/SlotView.java)

**核心特性**：
- 基于 Layout 类管理宫格布局
- 支持滚动（ScrollerHelper）
- 支持手势识别（GestureDetector）
- 支持 SlotRenderer 插件化渲染

**关键代码结构**：

```java
public class SlotView extends GLView {
    // 布局器
    private final Layout mLayout = new Layout();
    
    // 滚动辅助
    private final ScrollerHelper mScroller;
    
    // 手势识别
    private final GestureDetector mGestureDetector;
    
    // 渲染器
    private SlotRenderer mRenderer;
    
    // 渲染流程
    @Override
    protected void render(GLCanvas canvas) {
        super.render(canvas);
        if (mRenderer == null) return;
        mRenderer.prepareDrawing();
        
        // 渲染可见区域的 Slot
        int visibleStart = mLayout.getVisibleStart();
        int visibleEnd = mLayout.getVisibleEnd();
        for (int i = visibleStart; i < visibleEnd; i++) {
            Rect rect = mLayout.getSlotRect(i, mTempRect);
            int flags = mRenderer.renderSlot(canvas, i, pass, 
                                            rect.width(), rect.height());
        }
    }
}
```

### 4.4 AlbumSlotRenderer - 宫格渲染器

**文件位置**：[AlbumSlotRenderer.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/AlbumSlotRenderer.java)

```java
public class AlbumSlotRenderer extends AbstractSlotRenderer {
    private AlbumSlidingWindow mDataWindow;  // 滑动窗口缓存
    private final ColorTexture mWaitLoadingTexture;  // 等待占位符
    
    // 渲染单个 Slot
    @Override
    public int renderSlot(GLCanvas canvas, int index, int pass, 
                         int width, int height) {
        // 1. 获取数据
        AlbumSlidingWindow.AlbumEntry entry = mDataWindow.get(index);
        
        // 2. 渲染内容（照片/视频缩略图）
        Texture content = checkTexture(entry.content);
        if (content == null) {
            content = mWaitLoadingTexture;
        } else if (entry.isWaitDisplayed) {
            // 淡入动画
            content = new FadeInTexture(mPlaceholderColor, entry.bitmapTexture);
        }
        drawContent(canvas, content, width, height, entry.rotation);
        
        // 3. 渲染视频覆盖层（播放图标）
        if (entry.mediaType == MediaObject.MEDIA_TYPE_VIDEO) {
            drawVideoOverlay(canvas, width, height);
        }
        
        // 4. 渲染全景图标
        if (entry.isPanorama) {
            drawPanoramaIcon(canvas, width, height);
        }
        
        // 5. 渲染覆盖层（按下/选中状态）
        renderRequestFlags |= renderOverlay(canvas, index, entry, width, height);
        
        return renderRequestFlags;
    }
}
```

### 4.5 AlbumSlidingWindow - 数据滑动窗口

**作用**：
- 缓存可见区域附近的缩略图（默认 96 个）
- 后台加载缩略图
- 数据变化通知

### 4.6 触摸事件处理流程

```
用户触摸 (MotionEvent)
    ↓
SlotView.onTouch()
    ↓
GestureDetector (识别手势)
    ↓
SlotView.Listener 回调
    ↓
AlbumPage 处理
    ├── onDown(): 高亮按下的 Slot
    ├── onSingleTapUp(): 进入大图页
    └── onLongTap(): 进入多选模式
```

### 4.7 进入大图页流程

```
AlbumPage.onSingleTapUp()
    ↓
AlbumPage.pickPhoto()
    ↓
StateManager.startStateForResult(SinglePhotoPage)
    ↓
SinglePhotoPage.onCreate()
    ↓
SinglePhotoPage.onResume()
    ↓
PhotoView 渲染大图
```

---

## 大图页绘制分析

### 5.1 PhotoPage 整体架构

**文件位置**：[PhotoPage.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/PhotoPage.java)

```
PhotoPage (ActivityState)
    ├── mRootPane (GLView)
    │   └── mPhotoView (PhotoView)
    │       ├── mTileView (TileImageView) - 瓦片渲染高清图
    │       ├── mEdgeView (EdgeView) - 边缘效果
    │       └── mUndoBar (UndoBarView) - 撤销栏
    │
    ├── mModel (Model) - 数据模型
    ├── mBottomControls (PhotoPageBottomControls) - 底部控制栏
    └── mSelectionManager (SelectionManager)
```

### 5.2 PhotoView - 照片查看核心视图

**文件位置**：[PhotoView.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/PhotoView.java)

**核心功能**：
- 单张照片查看
- 左右滑动切换
- 缩放（双指）
- 拖拽（单指）
- 下拉删除
- 点击显示/隐藏 ActionBar

**内部数据结构**：

```java
public class PhotoView extends GLView {
    // 前后各缓存 3 张缩略图
    private static final int SCREEN_NAIL_MAX = 3;
    private final RangeArray<Picture> mPictures =
            new RangeArray<Picture>(-SCREEN_NAIL_MAX, SCREEN_NAIL_MAX);
    
    // 手势识别
    private final GestureRecognizer mGestureRecognizer;
    
    // 位置控制
    private final PositionController mPositionController;
    
    // 瓦片视图（高清图）
    private TileImageView mTileView;
}
```

### 5.3 PhotoView 渲染流程

```
render(GLCanvas canvas)
    ↓
1. 绘制当前照片（居中）
    ├── 如果未加载，显示占位符
    ├── 如果正在加载，显示加载动画
    └── 否则渲染 ScreenNail 或 TileView
    ↓
2. 绘制相邻照片（左右两边，带缩放和透明度）
    ↓
3. 绘制 UI 元素
    ├── 视频播放图标
    ├── 删除撤销栏
    └── 边缘效果
```

### 5.4 手势处理详解

**GestureRecognizer** 识别的手势：

| 手势 | 操作 |
|------|------|
| 单指拖拽 | 移动照片（已放大）或翻页 |
| 双指缩放 | 缩放照片 |
| 单指点击 | 切换 ActionBar 显示/隐藏 |
| 下拉 | 删除照片 |
| 快速滑动 | 翻页 |

**关键代码**：

```java
private class MyGestureListener extends GestureRecognizer.Listener {
    @Override
    public boolean onScroll(MotionEvent e1, MotionEvent e2,
                           float dx, float dy) {
        // 处理拖拽/滑动
    }
    
    @Override
    public boolean onScaleBegin(ScaleGestureDetector detector) {
        // 开始缩放
    }
    
    @Override
    public boolean onScale(ScaleGestureDetector detector) {
        // 缩放中
    }
}
```

### 5.5 TileImageView - 高清瓦片渲染

**设计思想**：对于大分辨率照片，不直接加载整张图，而是：
1. 先加载低分辨率缩略图（ScreenNail）
2. 根据当前视口，按需加载高分辨率瓦片
3. 渐进式渲染，提升性能

### 5.6 左右滑动切换照片

**实现原理**：
- 使用 `PositionController` 跟踪滑动位置
- 滑动时渲染当前、上一张、下一张照片
- 松手时根据速度和距离决定是回弹还是切换

```
滑动中 (onScroll):
  更新 PositionController
  invalidate() 触发重绘
  PhotoView.render() 绘制 3 张照片（带有偏移）

松手 (onUp):
  ScrollerHelper 计算惯性
  如果超过阈值，调用 mModel.moveTo() 切换
  否则回弹
```

### 5.7 与 AlbumPage 切换动画

**相册 → 大图**：
1. 记录点击的 Slot 在屏幕上的位置
2. 使用 `StateTransitionAnimation` 实现放大动画
3. 从 Slot 位置平滑过渡到全屏

**大图 → 相册**：
1. 记录当前照片在相册中的索引
2. 使用 `StateTransitionAnimation` 实现缩小动画
3. 从全屏平滑过渡回 Slot 位置

---

## 完整交互流程图

### 6.1 应用冷启动流程

```
用户点击图标
    ↓
ActivityManager 启动 GalleryActivity
    ↓
GalleryAppImpl.onCreate()
    ├── 初始化全局服务
    └── 初始化工具类
    ↓
GalleryActivity.onCreate()
    ├── setContentView(R.layout.main)
    ├── 初始化 GLRootView
    └── initializeByIntent()
        └── startDefaultPage()
            └── StateManager.startState(AlbumSetPage)
                ├── AlbumSetPage.onCreate()
                ├── AlbumSetPage.onResume()
                └── 显示相册列表
```

### 6.2 查看相册内容流程

```
AlbumSetPage 显示相册列表
    ↓
用户点击某个相册
    ↓
StateManager.startState(AlbumPage, data)
    ↓
AlbumPage.onCreate()
    ├── initializeViews()
    │   ├── 创建 SlotView
    │   └── 创建 AlbumSlotRenderer
    └── initializeData()
        ├── 获取 MediaSet
        └── 创建 AlbumDataLoader
    ↓
AlbumPage.onResume()
    ├── AlbumDataLoader.resume()
    ├── 开始加载媒体数据
    └── 渲染宫格
    ↓
用户滚动浏览
    ├── SlotView 滚动
    ├── AlbumSlidingWindow 按需加载
    └── AlbumSlotRenderer 渲染可见 Slot
```

### 6.3 查看大图完整流程

```
用户点击宫格中的照片
    ↓
AlbumPage.onSingleTapUp()
    ├── 记录点击位置
    └── AlbumPage.pickPhoto()
        ↓
StateManager.startStateForResult(SinglePhotoPage)
    ↓
SinglePhotoPage.onCreate()
    ├── 获取 Intent 数据（媒体路径）
    ├── 创建 PhotoView
    └── 设置 Model
    ↓
SinglePhotoPage.onResume()
    ├── PhotoView 开始加载
    ├── 显示缩略图
    └── 后台加载高清图
    ↓
高清图加载完成
    ├── TileImageView 渲染瓦片
    └── 用户可以缩放/拖拽
    ↓
用户左右滑动
    ├── PhotoView 切换相邻照片
    └── 预加载前后照片
    ↓
用户点击返回
    ├── SinglePhotoPage.onBackPressed()
    └── StateManager.finishState()
        ↓
AlbumPage.onResume()
    └── 恢复相册视图
```

### 6.4 照片编辑流程

```
PhotoPage 显示照片
    ↓
用户点击编辑按钮
    ↓
StateManager.startStateForResult(FilterShowPage)
    ↓
FilterShowActivity 启动
    ├── 加载原始照片
    ├── 显示编辑界面
    └── 应用滤镜/调整
    ↓
用户点击保存
    ├── ProcessingService 处理
    ├── 保存到新文件
    └── 刷新 MediaStore
    ↓
返回 PhotoPage
    └── 显示编辑后的照片
```

---

## 核心模块汇总

### 7.1 模块树

```
Gallery2
├── app/                          # 应用层
│   ├── GalleryActivity          # 主 Activity
│   ├── AbstractGalleryActivity  # Activity 基类
│   ├── StateManager             # 状态管理器
│   ├── ActivityState            # 页面基类
│   ├── AlbumSetPage             # 相册集页面
│   ├── AlbumPage                # 相册页面（宫格）
│   ├── PhotoPage                # 照片页面（大图）
│   └── ...
├── data/                         # 数据层
│   ├── DataManager              # 数据管理中枢
│   ├── MediaSet                 # 媒体集合
│   ├── MediaItem                # 媒体项
│   ├── LocalSource              # 本地媒体源
│   └── ...
├── ui/                           # UI 组件（基于 OpenGL）
│   ├── GLView                   # 视图基类
│   ├── GLRootView               # 根视图
│   ├── SlotView                 # 宫格视图
│   ├── PhotoView                # 照片视图
│   ├── AlbumSlotRenderer        # 宫格渲染器
│   └── ...
├── glrenderer/                   # OpenGL 渲染
│   ├── GLCanvas                 # 画布抽象
│   ├── GLES11Canvas             # GL 1.1 实现
│   ├── GLES20Canvas             # GL 2.0 实现
│   ├── Texture                  # 纹理
│   └── ...
├── filtershow/                   # 照片编辑
│   ├── FilterShowActivity       # 编辑器主界面
│   ├── filters/                 # 滤镜实现
│   ├── editors/                 # 编辑器
│   ├── pipeline/                # 处理管线
│   └── ...
├── ingest/                       # MTP 导入
│   ├── IngestActivity           # 导入界面
│   └── ...
├── jni/                          # JNI 原生代码
│   ├── filters/                 # C 语言滤镜
│   └── jni_egl_fence.cpp
├── jni_jpegstream/               # JPEG 流式处理
│   └── (C++ 代码)
└── gallerycommon/                # 公共库
    ├── exif/                    # EXIF 处理
    ├── jpegstream/              # JPEG 流
    └── common/                  # 通用工具
```

### 7.2 关键类索引

| 类名 | 职责 | 文件位置 |
|------|------|---------|
| `GalleryAppImpl` | 应用入口，全局服务 | [GalleryAppImpl.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryAppImpl.java) |
| `GalleryActivity` | 主界面，Intent 路由 | [GalleryActivity.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryActivity.java) |
| `StateManager` | 状态栈管理 | [StateManager.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/StateManager.java) |
| `ActivityState` | 页面基类 | [ActivityState.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/ActivityState.java) |
| `AlbumPage` | 宫格页 | [AlbumPage.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumPage.java) |
| `PhotoPage` | 大图页 | [PhotoPage.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/PhotoPage.java) |
| `SlotView` | 宫格视图 | [SlotView.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/SlotView.java) |
| `AlbumSlotRenderer` | 宫格渲染器 | [AlbumSlotRenderer.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/AlbumSlotRenderer.java) |
| `PhotoView` | 照片视图 | [PhotoView.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/PhotoView.java) |
| `GLCanvas` | OpenGL 画布 | [GLCanvas.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/GLCanvas.java) |
| `DataManager` | 数据管理 | 见 data/ 目录 |

---

## 总结

Gallery2 的设计亮点：

1. **自定义 UI 框架**：基于 OpenGL ES 的完整视图系统，比 Android 原生 View 更灵活、性能更好
2. **状态管理**：类似 Fragment 的栈式状态管理，但更轻量
3. **数据驱动**：MediaSet/MediaItem 抽象，支持多种数据源
4. **渐进式渲染**：TileImageView 按需加载瓦片，处理大照片
5. **滑动窗口缓存**：AlbumSlidingWindow 只缓存可见区域附近数据
6. **JNI 优化**：性能敏感的滤镜、JPEG 处理使用 C/C++

虽然这个项目已经不再维护，但其架构设计思想仍然值得学习和借鉴。
