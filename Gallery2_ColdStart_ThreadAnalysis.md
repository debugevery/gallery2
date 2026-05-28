# Gallery2 冷启动：完整线程协作分析

本文档详细分析冷启动过程中所有线程的角色、职责和协作机制。

---

## 一、所有线程汇总表

| 线程名/类型 | 优先级 | 线程创建方 | 主要职责 | 关键方法/类 |
|------------|-------|----------|---------|-----------|
| **Main Thread (UI Thread)** | THREAD_PRIORITY_DEFAULT | Android系统 | Activity生命周期、View创建、事件分发、Handler消息处理 | `GalleryActivity.onCreate()`、`GalleryAppImpl.onCreate()` |
| **GL Thread (Render Thread)** | THREAD_PRIORITY_DISPLAY | GLSurfaceView | OpenGL ES渲染循环、纹理上传、GLCanvas绘制 | `GLRootView.onDrawFrame()`、`TiledTexture.upload()` |
| **Reload Task Thread** | THREAD_PRIORITY_BACKGROUND | AlbumDataLoader | MediaSet数据同步、循环等待更新通知 | `ReloadTask.run()`、`MediaSet.reload()` |
| **ThreadPool Worker 1-8** | THREAD_PRIORITY_BACKGROUND | ThreadPool | 缩略图解码、Bitmap加载、磁盘IO | `Worker.run()`、`DecodeUtils.decodeThumbnail()` |
| **ContentObserver Thread** | SYSTEM | ContentResolver | MediaStore内容变化监听 | `ChangeNotifier`、`ContentListener` |

---

## 二、完整线程协作时序图

```
[Time: 0ms]
────────────────────────────────────────────────────────────────────────────
Main Thread              GL Thread          Reload Thread    Worker Threads
────────────────────────────────────────────────────────────────────────────
     │                        │                    │                    │
     ├─ Application.onCreate()                    │                    │
     │  ├─ init GalleryAppImpl                    │                    │
     │  └─ create ThreadPool (4 core threads)     │                    │
     │                        │                    │                    │
     ├─ Activity.onCreate()                        │                    │
     │  ├─ inflate layout                         │                    │
     │  ├─ create GLRootView                      │                    │
     │  │     └─ create GLSurfaceView              │                    │
     │  ├─ create DataManager                      │                    │
     │  └─ create AlbumPage                        │                    │
     │                        │                    │                    │
     ├─ Activity.onResume()                       │                    │
     │  ├─ AlbumPage.resume()                     │                    │
     │  │  ├─ AlbumDataLoader.resume() ──────────┐ │                    │
     │  │  │     └─ new ReloadTask().start()     │ │                    │
     │  │  │                                    │ │                    │
     │  │  └─ AlbumSlotRenderer.resume()         │ │                    │
     │  │     └─ AlbumSlidingWindow.resume()      │ │                    │
     │  └─ GLRootView.onResume()               │ │                    │
     │                        │                    │                    │
     │                        ├─ onSurfaceCreated()│                    │
     │                        │  └─ init EGL      │                    │
     │                        │                    │                    │
     │                        ├─ onSurfaceChanged()│                    │
     │                        │  └─ layout GLView │                    │
     │                        │                    │                    │
     │                        ├─ onDrawFrame() ──┐ │                    │
     │                        │                 │ │                    │
     │                        │                 │ │                    │
     │                        │                 │ │                    │
     │                        │                 │ │                    │

[Time: 25ms]
────────────────────────────────────────────────────────────────────────────
Main Thread              GL Thread          Reload Thread    Worker Threads
────────────────────────────────────────────────────────────────────────────
     │                        │                    │                    │
     │                        │                    │  ├─ ReloadTask.run() │
     │                        │                    │  └─ mSource.reload()│
     │                        │                    │                    │
     │                        │                    │  ├─ executeAndWait()│
     │  ├─ GetUpdateInfo.call()←──────────────┐ │                    │
     │  │     (在主线程同步执行)                │ │                    │
     │  │                                    │ │                    │
     │  └─ UpdateInfo返回────────────────────→│ │                    │
     │                        │                    │  │                 │
     │                        │                    │  └─ load MediaItems│
     │                        │                    │     (访问MediaStore) │
     │                        │                    │                    │
     │                        │                    │  ├─ executeAndWait()│
     │  ├─ UpdateContent.call()←───────────────────┐ │                    │
     │  │     (更新数据到主线程)                  │ │                    │
     │  │  ├─ update mData[]                      │ │                    │
     │  │  ├─ onSizeChanged()                      │ │                    │
     │  │  └─ onContentChanged()                   │ │                    │
     │  └─ UpdateContent完成──────────────────────→│ │                    │
     │                        │                    │                    │

[Time: 60ms - 首帧渲染]
────────────────────────────────────────────────────────────────────────────
Main Thread              GL Thread          Reload Thread    Worker Threads
────────────────────────────────────────────────────────────────────────────
     │                        │                    │                    │
     │  ├─ SlotView可见区域变化                 │                    │
     │  │  └─ onVisibleRangeChanged()           │                    │
     │  │     └─ AlbumSlidingWindow.setActiveWindow()│                    │
     │  │        └─ updateAllImageRequests()   │                    │
     │  │                                      │                    │
     │                        │                    │                    │
     │  ┌───────────────────────────────────────────────────────────────────┐
     │  │  发起缩略图请求: requestSlotImage() │                    │
     │  │  ├─ ThumbnailLoader.startLoad()     │                    │
     │  │  └─ ThreadPool.submit() ────────────────────────────────────────→│
     │  │                                      │                    │
     │                        │                    │                    │
     │                        │                    │                    │
     │                        │                    │  ┌─ Worker.run()    │
     │                        │                    │  │   ├─ MODE_CPU   │
     │                        │                    │  │   ├─ MediaItem.requestImage() │
     │                        │                    │  │   │   ├─ ImageCacheService.get() │
     │                        │                    │  │   │   │   └─ (磁盘缓存查找)  │
     │                        │                    │  │   │   └─ (未命中 → DecodeUtils.decode()) │
     │                        │                    │  │   └─ Bitmap解码 │
     │                        │                    │  │                  │
     │                        │                    │  └─ onFutureDone() │
     │                        │                    │      │              │
     │                        │                    │      └─ sendMessage(MSG_UPDATE_ENTRY) │
     │                        │                    │                    │
     │                        │  ┌─ SynchronizedHandler.handleMessage() │
     │                        │  │  └─ ThumbnailLoader.updateEntry()   │
     │                        │  │     ├─ TiledTexture创建             │
     │                        │  │     └─ addTexture()                │
     │                        │  │                                     │
     │                        │  └─ onDrawFrame()再次触发              │
     │                        │     ├─ uploadTextures()               │
     │                        │     │  └─ 每帧最多4ms上传              │
     │                        │     └─ AlbumSlotRenderer.renderSlot() │
     │                        │        └─ 绘制纹理到屏幕               │
     │                        │                    │                    │
     │                        │                    │                    │

[Time: 100ms - 首个缩略图显示]
────────────────────────────────────────────────────────────────────────────
Main Thread              GL Thread          Reload Thread    Worker Threads
────────────────────────────────────────────────────────────────────────────
     │                        │                    │                    │
     │                        │  ├─ 纹理已上传完成 │                    │
     │                        │  └─ 正常渲染循环   │                    │
     │                        │                    │                    │
     │                        │                    │                    │
     │                        │                    │  ├─ 活跃请求完成   │
     │                        │                    │  └─ requestNonactiveImages() │
     │                        │                    │    (开始加载非活跃区域) │
     │                        │                    │                    │
     │                        │                    │                    │
```

---

## 三、各线程详细分析

### 3.1 Main Thread (主线程/UI线程)

**文件**: 系统创建，通过`SynchronizedHandler`处理消息

**优先级**: `THREAD_PRIORITY_DEFAULT`

**创建时机**: 应用启动时由Android系统创建

**关键职责**:
| 阶段 | 职责 | 关键代码点 |
|------|------|----------|
| 初始化 | Application.onCreate() | `GalleryAppImpl.onCreate()` |
| Activity创建 | 布局创建、View初始化 | `GalleryActivity.onCreate()` |
| 页面跳转 | ActivityState管理 | `StateManager.startState()` |
| 消息处理 | Handler回调 | `SynchronizedHandler.dispatchMessage()` |
| 数据同步 | ReloadTask回调 | `GetUpdateInfo.call()`、`UpdateContent.call()` |
| 生命周期 | resume/pause/onBack | `AlbumPage.onResume()` |
| 事件分发 | 触摸、按键事件 | `GLRootView.dispatchTouchEvent()` |

**Handler消息类型**:
| msg.what | 来源 | 处理 |
|---------|------|------|
| MSG_RUN_OBJECT (3) | `executeAndWait()` | 执行Callable |
| MSG_LOAD_START (1) | ReloadTask | 通知加载开始 |
| MSG_LOAD_FINISH (2) | ReloadTask | 通知加载完成 |
| MSG_UPDATE_ENTRY (0) | ThumbnailLoader | 更新缩略图 |

---

### 3.2 GL Thread (渲染线程)

**文件**: [GLRootView.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/GLRootView.java)

**优先级**: `THREAD_PRIORITY_DISPLAY` (高优先级)

**创建时机**: `GLRootView.setRenderer()`后，GLSurfaceView内部创建

**关键职责**:

| 职责 | 方法 | 说明 |
|------|------|------|
| 渲染循环 | `onDrawFrame()` | 约60fps，每帧执行一次 |
| 纹理上传 | `TiledTexture.upload()` | 每帧最多4ms上传预算 |
| 绘制 | `GLCanvas.drawTexture()` | OpenGL ES绘制 |
| Handler消息 | `SynchronizedHandler.dispatchMessage()` | 带渲染锁的消息处理 |
| 布局 | `GLView.layout()` | 在首帧前和尺寸变化时执行 |

**核心渲染循环**:
```
onDrawFrame() {
    1. 检查是否需要布局 (FLAG_NEED_LAYOUT)
    2. 上传纹理 (4ms预算)
    3. mContentView.render() 绘制
       └─ SlotView.render()
          └─ AlbumSlotRenderer.renderSlot()
    4. 动画更新
    5. 空闲监听器回调
}
```

**线程同步机制**:
```java
// SynchronizedHandler保证渲染线程安全
public void dispatchMessage(Message message) {
    mRoot.lockRenderThread();  // 获取渲染锁
    try {
        super.dispatchMessage(message);  // 执行消息
    } finally {
        mRoot.unlockRenderThread();  // 释放锁
    }
}

// GLRootView内部锁
private final ReentrantLock mRenderLock = new ReentrantLock();
```

---

### 3.3 Reload Task Thread (数据同步线程)

**文件**: [AlbumDataLoader.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumDataLoader.java#L338-L396)

**优先级**: `THREAD_PRIORITY_BACKGROUND`

**创建时机**: `AlbumDataLoader.resume()` → `new ReloadTask().start()`

**核心循环**:
```
ReloadTask.run() {
    while (mActive) {
        synchronized (this) {
            if (mActive && !mDirty && updateComplete) {
                wait();  // 等待内容变化通知
                continue;
            }
            mDirty = false;
        }
        
        1. mSource.reload()  // 同步MediaSet
        2. executeAndWait(new GetUpdateInfo())  // 主线程同步执行
        3. info.items = mSource.getMediaItem()  // 后台加载数据
        4. executeAndWait(new UpdateContent())  // 主线程同步更新
    }
}
```

**状态说明**:
| 状态 | 说明 |
|------|------|
| mActive | true=运行中，false=终止 |
| mDirty | true=有新内容需要同步 |
| updateComplete | true=本次更新完整 |
| mIsLoading | true=正在加载中 |

**与主线程同步**:
```java
private <T> T executeAndWait(Callable<T> callable) {
    FutureTask<T> task = new FutureTask<T>(callable);
    // 发送到主线程Handler
    mMainHandler.sendMessage(MSG_RUN_OBJECT, task);
    // 阻塞等待主线程执行完成
    return task.get();
}
```

---

### 3.4 ThreadPool Worker Threads (缩略图解码线程)

**文件**: [ThreadPool.java](file:///d:/code/gallery2/Gallery2/gallerycommon/src/com/android/gallery3d/util/ThreadPool.java)

**线程数量**:
- 核心线程数 (CORE_POOL_SIZE): 4
- 最大线程数 (MAX_POOL_SIZE): 8
- 空闲存活时间 (KEEP_ALIVE_TIME): 10秒

**优先级**: `THREAD_PRIORITY_BACKGROUND`

**资源限制**:
| 资源类型 | 最大并发数 | 用途 |
|---------|-----------|------|
| CPU | 2 | 缩略图解码 |
| NETWORK | 2 | 网络IO |

**Job执行流程**:
```java
Worker.run() {
    // 默认是CPU模式
    if (setMode(MODE_CPU)) {  // 申请CPU资源 (最多2个并发)
        result = mJob.run(this);  // 执行Job
    }
    setMode(MODE_NONE);  // 释放资源
    mListener.onFutureDone(this);  // 回调
}
```

**资源获取机制**:
```java
private boolean acquireResource(ResourceCounter counter) {
    while (true) {
        synchronized (counter) {
            if (counter.value > 0) {
                counter.value--;  // 获取资源
                break;
            } else {
                counter.wait();  // 资源不足，等待
            }
        }
    }
    return true;
}
```

**JobLimiter限制**:
```java
// AlbumSlidingWindow内部
mThreadPool = new JobLimiter(..., 2);  // 限制最多2个并发缩略图加载
```

---

### 3.5 ContentObserver Thread (MediaStore变化监听)

**文件**: [ChangeNotifier.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/data/ChangeNotifier.java)

**优先级**: 系统管理的线程

**职责**:
- 监听MediaStore数据库变化
- 回调 `ContentListener.onContentDirty()`
- 唤醒ReloadTask进行重新同步

---

## 四、线程间通信机制详解

### 4.1 Handler消息机制 (GL线程)

**SynchronizedHandler特点**:
- 绑定到GLRootView
- 消息处理时持有渲染锁
- 保证与渲染循环互斥

**消息发送流程**:
```
Worker线程 (解码完成)
    ↓
BitmapLoader.onLoadComplete()
    ↓
mHandler.obtainMessage(MSG_UPDATE_ENTRY, this).sendToTarget()
    ↓
SynchronizedHandler.dispatchMessage()  [GL线程]
    ↓
mRoot.lockRenderThread()  // 获取渲染锁
    ↓
ThumbnailLoader.updateEntry()
    ↓
mRoot.unlockRenderThread()  // 释放锁
```

### 4.2 FutureTask同步机制 (主线程+Reload线程)

```
ReloadTask线程                    Main线程
    │                              │
    ├─ executeAndWait()            │
    │   └─ new FutureTask()        │
    │                              │
    │   └─ sendMessage() ────────→│
    │                              │
    │   (wait阻塞)                 │
    │                              ├─ dispatchMessage()
    │                              │  └─ FutureTask.run()
    │                              │     └─ callable.call()
    │                              │        └─ GetUpdateInfo / UpdateContent
    │                              │
    │←─────────────────────────────│
    │                              │
    │   (唤醒)                     │
    │                              │
    └─ task.get() 返回结果         │
```

### 4.3 Object.wait() / notifyAll() (Reload线程)

```
ReloadTask.run() {
    synchronized (this) {
        while (mActive && !mDirty) {
            wait();  // 等待内容变化
        }
    }
}

notifyDirty() {
    synchronized (this) {
        mDirty = true;
        notifyAll();  // 唤醒等待线程
    }
}
```

---

## 五、冷启动完整线程协作流程

### 5.1 阶段一：初始化 (0ms - 20ms)

```
[Main Thread]
├─ Application.onCreate()
│  ├─ init ThreadPool (4 core threads)
│  ├─ init GalleryUtils
│  └─ init DataManager
│
└─ GalleryActivity.onCreate()
   ├─ setContentView() → 创建GLRootView
   └─ StateManager.startState(AlbumPage)
```

### 5.2 阶段二：页面激活 (20ms - 30ms)

```
[Main Thread]
├─ Activity.onResume()
│  ├─ GalleryAppImpl.getImageCacheService()
│  │     └─ (首次初始化：可能有磁盘IO)
│  │
│  ├─ AlbumPage.onResume()
│  │  ├─ AlbumDataLoader.resume()
│  │  │  └─ new ReloadTask().start()
│  │  │
│  │  └─ AlbumSlotRenderer.resume()
│  │     └─ AlbumSlidingWindow.resume()
│  │
│  └─ GLRootView.onResume()
│     └─ start GL渲染循环
│
[GL Thread 启动]
├─ onSurfaceCreated()
├─ onSurfaceChanged()
└─ onDrawFrame() - 首帧开始
```

### 5.3 阶段三：数据加载 (30ms - 60ms)

```
[Reload Task Thread]
├─ ReloadTask.run()
│  ├─ mSource.reload()  // 同步MediaSet
│  │
│  ├─ executeAndWait(GetUpdateInfo)
│  │  │
│  │  └─ [Main Thread]
│  │     └─ GetUpdateInfo.call()
│  │
│  ├─ info.items = mSource.getMediaItem()
│  │  └─ (访问MediaStore数据库)
│  │
│  └─ executeAndWait(UpdateContent)
│     │
│     └─ [Main Thread]
│        └─ UpdateContent.call()
│           ├─ 更新 mData[]
│           ├─ onSizeChanged()
│           └─ onContentChanged()
```

### 5.4 阶段四：缩略图显示 (60ms - 100ms)

```
[Main Thread]  - 首次渲染
├─ SlotView布局完成
├─ onVisibleRangeChanged()
├─ AlbumSlidingWindow.setActiveWindow()
└─ updateAllImageRequests()

[Worker Threads]  - 解码缩略图
├─ JobLimiter限制并发为2
├─ Worker1: 解码缩略图1
│  ├─ ImageCacheService.getImageData()
│  │  ├─ 命中磁盘缓存 → 直接decode
│  │  └─ 未命中 → 从文件decode
│  └─ BitmapLoader.onFutureDone()
│
└─ Worker2: 解码缩略图2

[GL Thread]  - 上传与渲染
├─ SynchronizedHandler.handleMessage()
│  └─ ThumbnailLoader.updateEntry()
│     ├─ new TiledTexture(bitmap)
│     └─ mTileUploader.addTexture()
│
└─ onDrawFrame()
   ├─ uploadTextures() - 上传纹理
   │  └─ 4ms预算限制
   │
   └─ AlbumSlotRenderer.renderSlot()
      └─ 首个缩略图显示
```

### 5.5 阶段五：持续工作 (100ms+ )

```
[GL Thread]  - 持续渲染循环
├─ onDrawFrame() - 约16ms每帧
│  ├─ uploadTextures()
│  ├─ render()
│  └─ 动画更新
│
[Worker Threads]  - 持续加载
├─ 活跃区域缩略图完成
├─ requestNonactiveImages()
│  └─ 开始加载非活跃区域
│
└─ 循环加载...

[Reload Task Thread]  - 同步
├─ 等待MediaStore变化
├─ 有新图片 → notifyDirty()
│  └─ 重新同步数据
│
└─ 无变化 → wait()等待
```

---

## 六、线程优先级设计

| 线程 | 优先级 | 原因 |
|------|-------|------|
| GL Thread | THREAD_PRIORITY_DISPLAY | 高优先级，保证渲染流畅不卡顿 |
| Main Thread | THREAD_PRIORITY_DEFAULT | UI交互响应及时 |
| Worker Threads | THREAD_PRIORITY_BACKGROUND | 解码是后台任务，不抢占UI资源 |
| Reload Task Thread | THREAD_PRIORITY_BACKGROUND | 数据同步不紧急，后台处理 |
| ContentObserver | SYSTEM | 系统决定，监听MediaStore |

---

## 七、关键线程同步点

### 7.1 资源限制同步

**位置**: [ThreadPool.Worker.acquireResource()](file:///d:/code/gallery2/Gallery2/gallerycommon/src/com/android/gallery3d/util/ThreadPool.java#L230-L259)

```
多个Worker同时申请CPU资源
    ↓
CPU资源上限: 2
    ↓
Worker 3 必须等待 Worker 1或2 完成并释放资源
    ↓
wait() - notify() 机制
```

### 7.2 渲染线程锁

**位置**: [GLRootView.lockRenderThread()](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/GLRootView.java)

```
渲染循环与Handler消息处理互斥
    ↓
使用 ReentrantLock
    ↓
保证同一时间只有一个在访问GL状态
```

### 7.3 FutureTask阻塞等待

**位置**: [AlbumDataLoader.executeAndWait()](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumDataLoader.java#L222-L233)

```
Reload线程必须等待主线程执行完更新才能继续
    ↓
FutureTask.get()阻塞
    ↓
防止数据竞争
```

### 7.4 ReloadTask唤醒机制

**位置**: [ReloadTask.notifyDirty()](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumDataLoader.java#L387-L390)

```
ContentObserver监听到变化
    ↓
notifyDirty()
    ↓
mDirty=true + notifyAll()
    ↓
Reload线程从wait()唤醒
```

---

## 八、内存可见性保证

| 机制 | 用途 | 关键代码 |
|------|------|---------|
| `volatile` | 简单状态标记 | `mActive`、`mDirty`、`mRenderRequested` |
| `synchronized` | 复杂对象共享 | `mData[]` 访问 |
| `FutureTask` | 任务结果传递 | `executeAndWait()` 返回值 |
| `Handler` | 线程间消息 | Message对象在内存屏障后可见 |

---

## 九、性能优化设计

### 9.1 并发控制

| 优化 | 作用 |
|------|------|
| JobLimiter | 缩略图加载最多2个并发，避免太多IO导致卡顿 |
| CPU资源计数器 | 最多2个解码任务，避免CPU过热 |
| 线程池大小 | 核心4个，最大8个，平衡吞吐量和资源占用 |

### 9.2 任务调度

| 优化 | 作用 |
|------|------|
| 活跃区域优先 | 先加载可见区域，提升用户感知 |
| 非活跃区域后台加载 | 活跃区域完成后再加载缓冲区，不抢资源 |
| 滑动时取消请求 | 快速滑动时取消旧请求，避免浪费 |

### 9.3 渲染性能

| 优化 | 作用 |
|------|------|
| 纹理上传预算 | 每帧最多4ms上传，避免拖慢渲染 |
| TiledTexture | 大图分片，降低内存占用 |
| OpenGL ES | GPU加速，高性能渲染 |

---

## 十、故障情况处理

### 10.1 加载失败处理

```
ReloadTask检测到加载失败
    ↓
mFailedVersion = version
    ↓
暂停加载，等待用户操作
```

### 10.2 任务取消处理

```
Worker任务被取消
    ↓
mIsCancelled = true
    ↓
调用 CancelListener
    ↓
BitmapFactory.Options.requestCancelDecode()
    ↓
解码提前终止
```

### 10.3 页面退出清理

```
AlbumPage.pause()
    ↓
AlbumDataLoader.pause()
    ├─ ReloadTask.terminate()
    │  └─ mActive=false, notifyAll()
    │
    └─ 移除ContentListener
    │
AlbumSlotRenderer.pause()
    └─ AlbumSlidingWindow.pause()
       └─ 释放所有纹理、取消所有加载任务
```

---

## 十一、关键文件索引

| 功能 | 文件 | 主要类 |
|------|------|--------|
| 线程池 | `ThreadPool.java` | `ThreadPool`、`Worker` |
| 数据同步 | `AlbumDataLoader.java` | `ReloadTask`、`GetUpdateInfo`、`UpdateContent` |
| 渲染线程 | `GLRootView.java` | `GLRootView`、`SynchronizedHandler` |
| 缩略图加载 | `AlbumSlidingWindow.java` | `ThumbnailLoader`、`JobLimiter` |
| 图像解码 | `DecodeUtils.java` | 解码逻辑 |
| 缓存服务 | `ImageCacheService.java` | 磁盘缓存 |
| Bitmap池 | `GalleryBitmapPool.java` | 内存复用 |

---

## 总结

Gallery2的线程设计是一个非常优秀的多线程架构：
1. **任务分离清晰**: UI、渲染、数据、IO各司其职
2. **优先级明确**: 保证渲染和UI响应优先
3. **资源控制严格**: 通过计数器和JobLimiter避免资源浪费
4. **同步机制完善**: 多种同步手段配合使用，避免竞争

这种设计使得即使在处理数千张图片时，Gallery2仍然能够保持流畅的用户体验。
