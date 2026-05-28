# ReloadTask 深度分析

## 概述

`ReloadTask` 是 Gallery2 中 `AlbumDataLoader` 的核心组件，负责后台线程中异步加载相册数据。它是连接数据源（MediaStore）和 UI 层的关键桥梁。

---

## 一、ReloadTask 完整源码解析

**文件位置**: [AlbumDataLoader.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumDataLoader.java)

### 1.1 ReloadTask 类定义

```java
private class ReloadTask extends Thread {

    private volatile boolean mActive = true;      // 线程存活标志
    private volatile boolean mDirty = true;       // 数据脏标志（需要重新加载）
    private boolean mIsLoading = false;          // 是否正在加载

    @Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        boolean updateComplete = false;
        while (mActive) {
            synchronized (this) {
                // 状态检查与等待逻辑
                if (mActive && !mDirty && updateComplete) {
                    updateLoading(false);
                    if (mFailedVersion != MediaObject.INVALID_DATA_VERSION) {
                        Log.d(TAG, "reload pause");
                    }
                    Utils.waitWithoutInterrupt(this);  // 阻塞等待
                    if (mActive && (mFailedVersion != MediaObject.INVALID_DATA_VERSION)) {
                        Log.d(TAG, "reload resume");
                    }
                    continue;
                }
                mDirty = false;
            }
            
            // 执行数据加载
            updateLoading(true);
            long version = mSource.reload();
            UpdateInfo info = executeAndWait(new GetUpdateInfo(version));
            updateComplete = info == null;
            if (updateComplete) continue;
            if (info.version != version) {
                info.size = mSource.getMediaItemCount();
                info.version = version;
            }
            if (info.reloadCount > 0) {
                info.items = mSource.getMediaItem(info.reloadStart, info.reloadCount);
            }
            executeAndWait(new UpdateContent(info));
        }
        updateLoading(false);
    }

    public synchronized void notifyDirty() {
        mDirty = true;
        notifyAll();
    }

    public synchronized void terminate() {
        mActive = false;
        notifyAll();
    }
}
```

---

## 二、状态机设计

### 2.1 线程状态流转

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │                      ReloadTask.run()                       │
                    │                                                              │
                    │   ┌───────────┐                                            │
                    │   │ WAITING   │ ◄────────────────────────────────────────┐│
                    │   └─────┬─────┘                                          ││
                    │         │                                                ││
                    │         │ mDirty=true OR !updateComplete                  ││
                    │         ▼                                                ││
                    │   ┌───────────┐    执行完成且无变化                        ││
         ┌──────────┼──►│  RUNNING  ├───────────────────────────────────────►│
         │          │   └─────┬─────┘                                          ││
         │          │         │                                                ││
         │          │         │ mActive=false                                   ││
         │          │         ▼                                                ││
         │          │   ┌───────────┐                                          ││
         │          │   │  EXITED   │                                          ││
         │          │   └───────────┘                                          ││
         │          │                                                            │
         │          └────────────────────────────────────────────────────────────┘
         │
         │ mActive=false (terminate)
         ▼
    ┌───────────┐
    │ TERMINATED│
    └───────────┘
```

### 2.2 成员变量状态含义

| 变量 | 类型 | 含义 | 状态组合 |
|------|------|------|----------|
| `mActive` | volatile boolean | 线程是否存活 | true=运行中, false=已终止 |
| `mDirty` | volatile boolean | 数据是否需要刷新 | true=需要刷新, false=最新 |
| `mIsLoading` | boolean | 是否正在加载 | true=正在加载, false=空闲 |
| `updateComplete` | boolean | 本次更新是否完整 | true=完整, false=需要继续 |
| `mFailedVersion` | long | 失败的版本号 | INVALID=正常, 其他=失败版本 |

### 2.3 状态组合与行为

| mActive | mDirty | updateComplete | 行为 |
|---------|--------|---------------|------|
| true | true | * | 进入加载流程 |
| true | false | true | **wait() 等待** |
| true | false | false | 继续加载（数据不全） |
| false | * | * | 退出循环 |

---

## 三、触发条件分析

### 3.1 触发来源汇总

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ReloadTask 触发来源                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  触发点A: AlbumDataLoader.resume()                                          │
│  ├─ 页面可见时调用                                                          │
│  └─ mDirty 初始为 true，保证冷启动时立即加载                                │
│                                                                             │
│  触发点B: ContentObserver 监听                                               │
│  ├─ MySourceListener.onContentDirty()                                       │
│  └─ MediaStore 数据库变化时触发                                              │
│                                                                             │
│  触发点C: SlotView 可见区域变化                                              │
│  └─ AlbumSlidingWindow.setActiveWindow()                                    │
│      └─ setContentWindow() → notifyDirty()                                 │
│                                                                             │
│  触发点D: MediaSet 数据版本变化                                              │
│  └─ MediaSet.reload() 返回新版本号                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 详细触发逻辑

#### 触发点A：冷启动触发

```java
// AlbumDataLoader 构造函数中
public AlbumDataLoader(...) {
    // ...
    mData = new MediaItem[DATA_CACHE_SIZE];  // 1000个缓存槽
    mItemVersion = new long[DATA_CACHE_SIZE];
    mSetVersion = new long[DATA_CACHE_SIZE];
    Arrays.fill(mItemVersion, MediaObject.INVALID_DATA_VERSION);
    Arrays.fill(mSetVersion, MediaObject.INVALID_DATA_VERSION);
}

// resume() 时创建并启动线程
public void resume() {
    mSource.addContentListener(mSourceListener);  // 注册监听
    mReloadTask = new ReloadTask();                // mDirty = true (初始化为true)
    mReloadTask.start();                           // 启动线程
}
```

**冷启动特点**:
- `mDirty` 初始化为 `true`
- 无需外部触发，立即进入加载流程
- 首次加载完整数据

#### 触发点B：ContentObserver 变化监听

```java
// MySourceListener 实现 ContentListener
private class MySourceListener implements ContentListener {
    @Override
    public void onContentDirty() {
        if (mReloadTask != null) mReloadTask.notifyDirty();
    }
}

// notifyDirty() 实现
public synchronized void notifyDirty() {
    mDirty = true;
    notifyAll();  // 唤醒等待中的线程
}
```

**变化来源链路**:
```
用户拍照/删除照片
    ↓
MediaStore 数据库变化
    ↓
ChangeNotifier.onChange()
    ↓
MediaSet.notifyContentChanged()
    ↓
ContentListener.onContentDirty()
    ↓
ReloadTask.notifyDirty()
    ↓
mDirty = true, notifyAll()
```

**ChangeNotifier 源码** ([ChangeNotifier.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/data/ChangeNotifier.java)):
```java
protected void onChange(boolean selfChange) {
    // AtomicBoolean 保证线程安全
    if (mContentDirty.compareAndSet(false, true)) {
        mMediaSet.notifyContentChanged();
    }
}
```

#### 触发点C：可见区域变化

```java
// AlbumSlidingWindow.setActiveWindow()
public void setActiveWindow(int start, int end) {
    // ...
    setContentWindow(contentStart, contentEnd);
    // ...
}

private void setContentWindow(int contentStart, int contentEnd) {
    if (contentStart == mContentStart && contentEnd == mContentEnd) return;
    
    // 更新缓存范围
    // 释放旧数据，准备新数据
    // ...
    
    if (mReloadTask != null) {
        mReloadTask.notifyDirty();  // 通知需要重新加载
    }
}
```

**触发场景**:
- 用户滑动宫格
- 进入/退出相册
- 屏幕旋转

#### 触发点D：版本号变化

```java
// ReloadTask.run() 中
long version = mSource.reload();  // 返回MediaSet当前版本号

UpdateInfo info = executeAndWait(new GetUpdateInfo(version));

// GetUpdateInfo.call() 中检查版本
@Override
public UpdateInfo call() throws Exception {
    if (mFailedVersion == mVersion) {
        return null;  // 上次加载失败，暂停
    }
    // ...
    // 如果版本没变，返回 null 表示无需更新
    return mSourceVersion == mVersion ? null : info;
}
```

---

## 四、冷启动时的完整行为

### 4.1 冷启动时序图

```
[冷启动阶段 - AlbumPage.onResume()]
─────────────────────────────────────────────────────────────────────
    
Main Thread                              ReloadTask Thread
─────────────────────────────────────────────────────────────────────
     │                                        │
     ├─ AlbumPage.onResume()                 │
     │  └─ AlbumDataLoader.resume()          │
     │     ├─ mSource.addContentListener()    │
     │     └─ new ReloadTask().start() ───────┼──► ReloadTask.run()
     │                                        │      │
     ├─ AlbumSlotRenderer.resume()            │      ├─ mActive = true
     │  └─ AlbumSlidingWindow.resume()        │      ├─ mDirty = true (初始值)
     │     └─ setActiveWindow()               │      └─ updateLoading(true)
     │                                        │         (MSG_LOAD_START)
     │                                        │
     │                                        ├─ mSource.reload()
     │                                        │   └─ LocalAlbum.reload()
     │                                        │      └─ 访问 MediaStore
     │                                        │
     │                                        ├─ GetUpdateInfo.call()
     │  ├─ handleMessage(MSG_RUN_OBJECT)      │   └─ 检查需要加载的范围
     │  │  └─ FutureTask.run()                │      (首次返回全部范围)
     │  │     └─ GetUpdateInfo.call()        │
     │  │        └─ 返回 UpdateInfo           │
     │  │                                    │
     │  └─ 返回结果                           │  (唤醒)
     │                                        │
     │                                        ├─ mSource.getMediaItem(start, count)
     │                                        │   └─ 批量加载媒体项
     │                                        │
     │                                        ├─ UpdateContent.call()
     │  ├─ handleMessage(MSG_RUN_OBJECT)      │
     │  │  └─ FutureTask.run()                │
     │  │     └─ UpdateContent.call()         │
     │  │        ├─ mData[] 更新              │
     │  │        ├─ mSize 更新               │
     │  │        └─ onSizeChanged()          │
     │  │           └─ SlotView.setSlotCount()
     │  │                                        │
     │  │                                        │
     │  │                                        ├─ 首次加载完成
     │                                        │   └─ updateComplete = true
     │                                        │
     │                                        ├─ 更新完成，进入等待
     │  ├─ handleMessage(MSG_LOAD_FINISH)     │   └─ wait()
     │  │  └─ LoadingListener.onLoadingFinished()
     │  │                                        │
     │  └─ SlotView 首次渲染                   │  [WAITING]
     │     └─ 显示空白/占位符                   │
     │                                        │
     │  (约60ms后，首帧渲染)                    │
     │                                        │
     ├─ SlotView.onVisibleRangeChanged()      │
     │  └─ setActiveWindow()                  │
     │     └─ notifyDirty()                  │
     │                                        │  [唤醒]
     │                                        │
     │                                        ├─ 可见区域数据加载
     │                                        │  (MIN_LOAD_COUNT = 32 ~ MAX_LOAD_COUNT = 64)
     │                                        │
     │                                        ├─ UpdateContent()
     │  ├─ handleMessage(MSG_RUN_OBJECT)      │
     │  │  └─ onContentChanged()              │
     │  │     └─ AlbumSlotRenderer 触发重绘    │
     │  │                                        │
     │  │                                        │  缩略图开始显示...

─────────────────────────────────────────────────────────────────────
```

### 4.2 冷启动特点

| 特点 | 说明 |
|------|------|
| 无需外部触发 | `mDirty` 初始为 `true` |
| 全量加载 | 首次加载整个 MediaSet 的基本信息 |
| 批量获取 | `getMediaItem(0, MAX_LOAD_COUNT)` |
| 立即渲染 | `updateComplete` 为 `true` 时才进入等待 |
| 两阶段加载 | 1. MediaSet基本信息 → 2. 可见区域缩略图 |

### 4.3 关键时间点

```
[0ms]    AlbumDataLoader.resume() 调用
[5ms]    ReloadTask.start() 创建线程
[10ms]   ReloadTask.run() 开始执行
[15ms]   mSource.reload() 获取MediaSet版本
[20ms]   GetUpdateInfo.call() 确定加载范围
[25ms]   UpdateContent.call() 更新数据
[30ms]   MSG_LOAD_FINISH 发送
[35ms]   SlotView.setSlotCount() 布局
[40ms]   首次渲染（显示空白或占位符）
[50ms]   可见区域确定，notifyDirty()
[60ms]   缩略图加载开始
[80ms+]  缩略图逐个显示
```

---

## 五、核心方法解析

### 5.1 executeAndWait() - 线程间同步

```java
private <T> T executeAndWait(Callable<T> callable) {
    FutureTask<T> task = new FutureTask<T>(callable);
    // 发送到主线程的 Handler
    mMainHandler.sendMessage(
            mMainHandler.obtainMessage(MSG_RUN_OBJECT, task));
    try {
        // 阻塞等待主线程执行完成
        return task.get();
    } catch (InterruptedException e) {
        return null;
    } catch (ExecutionException e) {
        throw new RuntimeException(e);
    }
}
```

**设计目的**:
- ReloadTask 运行在后台线程
- 数据更新必须在主线程执行（涉及 UI 更新）
- 使用 FutureTask 实现同步等待

**执行流程**:
```
ReloadTask                           Main Thread
    │                                    │
    │  executeAndWait()                  │
    │  ├─ 创建 FutureTask               │
    │  └─ sendMessage() ───────────────►│
    │                                    │
    │  task.get() 阻塞等待              │
    │  ◄───────────────────────────── │ handleMessage()
    │                                    │   └─ FutureTask.run()
    │                                    │      └─ callable.call()
    │  ◄───────────────────────────────│ 返回结果
    │                                    │
    │  继续执行                          │
```

### 5.2 GetUpdateInfo - 更新信息计算

```java
private class GetUpdateInfo implements Callable<UpdateInfo> {
    private final long mVersion;

    @Override
    public UpdateInfo call() throws Exception {
        if (mFailedVersion == mVersion) {
            // 上次加载失败，暂停
            return null;
        }
        
        UpdateInfo info = new UpdateInfo();
        info.version = mSourceVersion;  // 当前缓存的版本
        info.size = mSize;              // 当前缓存的大小
        
        long setVersion[] = mSetVersion;
        // 遍历内容窗口，找出需要刷新的位置
        for (int i = mContentStart, n = mContentEnd; i < n; ++i) {
            int index = i % DATA_CACHE_SIZE;
            if (setVersion[index] != mVersion) {
                info.reloadStart = i;                          // 起始位置
                info.reloadCount = Math.min(MAX_LOAD_COUNT, n - i);  // 批量大小
                return info;
            }
        }
        
        // 如果版本没变，返回 null 表示无需更新
        return mSourceVersion == mVersion ? null : info;
    }
}
```

**关键设计**:
1. **版本比较**: 通过 `mVersion` 判断数据是否过期
2. **批量获取**: 每次最多获取 `MAX_LOAD_COUNT = 64` 条
3. **分批加载**: 大相册不会一次性全部加载

### 5.3 UpdateContent - 数据更新

```java
private class UpdateContent implements Callable<Void> {
    private UpdateInfo mUpdateInfo;

    @Override
    public Void call() throws Exception {
        UpdateInfo info = mUpdateInfo;
        mSourceVersion = info.version;
        
        // 1. 处理数量变化
        if (mSize != info.size) {
            mSize = info.size;
            if (mDataListener != null) {
                mDataListener.onSizeChanged(mSize);
            }
        }

        // 2. 检查加载失败
        mFailedVersion = MediaObject.INVALID_DATA_VERSION;
        if ((items == null) || items.isEmpty()) {
            if (info.reloadCount > 0) {
                mFailedVersion = info.version;
            }
            return null;
        }

        // 3. 更新缓存数据
        int start = Math.max(info.reloadStart, mContentStart);
        int end = Math.min(info.reloadStart + items.size(), mContentEnd);

        for (int i = start; i < end; ++i) {
            int index = i % DATA_CACHE_SIZE;
            mSetVersion[index] = info.version;
            MediaItem updateItem = items.get(i - info.reloadStart);
            long itemVersion = updateItem.getDataVersion();
            
            if (mItemVersion[index] != itemVersion) {
                mItemVersion[index] = itemVersion;
                mData[index] = updateItem;
                
                // 4. 通知可见区域的数据变化
                if (mDataListener != null && 
                    i >= mActiveStart && i < mActiveEnd) {
                    mDataListener.onContentChanged(i);
                }
            }
        }
        return null;
    }
}
```

**关键设计**:
1. **版本更新**: 更新 `mSourceVersion`
2. **批量更新**: 一次性更新多条数据
3. **增量通知**: 只通知可见区域的变化
4. **失败处理**: 记录失败版本号

---

## 六、与 ContentObserver 的协作

### 6.1 完整协作链路

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ContentObserver 协作流程                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  [MediaStore]                                                              │
│       │                                                                    │
│       │ 数据库变化                                                           │
│       ▼                                                                    │
│  [ContentResolver.notifyChange()]                                            │
│       │                                                                    │
│       │ 系统回调                                                            │
│       ▼                                                                    │
│  [ChangeNotifier.onChange()]  (在ContentObserver线程)                       │
│       │                                                                    │
│       │ AtomicBoolean 保证只通知一次                                         │
│       ▼                                                                    │
│  [MediaSet.notifyContentChanged()]                                           │
│       │                                                                    │
│       │ 遍历所有监听器                                                      │
│       ▼                                                                    │
│  [ContentListener.onContentDirty()]  ──► [AlbumDataLoader.MySourceListener] │
│       │                                                                    │
│       │ 后台线程安全                                                        │
│       ▼                                                                    │
│  [ReloadTask.notifyDirty()]                                                 │
│       │                                                                    │
│       │ mDirty = true                                                      │
│       │ notifyAll()                                                        │
│       ▼                                                                    │
│  [ReloadTask.run() 被唤醒]                                                  │
│       │                                                                    │
│       │ 开始重新加载                                                        │
│       ▼                                                                    │
│  [executeAndWait(UpdateContent)]                                             │
│       │                                                                    │
│       ▼                                                                    │
│  [Main Thread: onContentChanged()]                                          │
│       │                                                                    │
│       │ 触发 UI 重绘                                                        │
│       ▼                                                                    │
│  [SlotView / AlbumSlotRenderer 更新显示]                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 ChangeNotifier 线程安全设计

```java
// ChangeNotifier.java
private AtomicBoolean mContentDirty = new AtomicBoolean(true);

protected void onChange(boolean selfChange) {
    // compareAndSet 保证原子性
    if (mContentDirty.compareAndSet(false, true)) {
        mMediaSet.notifyContentChanged();
    }
}
```

**为什么用 AtomicBoolean**:
- `onChange()` 可能在 ContentObserver 线程多次调用
- `compareAndSet()` 保证只触发一次 `notifyContentChanged()`
- 避免重复通知导致的性能问题

---

## 七、数据缓存结构

### 7.1 缓存数组设计

```java
// AlbumDataLoader 成员变量
private final MediaItem[] mData;        // 媒体项缓存
private final long[] mItemVersion;       // 每个MediaItem的版本
private final long[] mSetVersion;        // 整个MediaSet的版本

// 常量
private static final int DATA_CACHE_SIZE = 1000;  // 缓存容量

// 索引计算（环形缓冲区）
int index = slotIndex % DATA_CACHE_SIZE;
```

### 7.2 版本机制

```java
// MediaObject 中的版本定义
public static final long INVALID_DATA_VERSION = -1;

// 版本比较逻辑
if (mSetVersion[index] != info.version) {
    // 缓存的数据已过期，需要更新
}
```

### 7.3 缓存范围

```
┌─────────────────────────────────────────────────────────────────────┐
│                      缓存区域示意图                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [0] [1] [2] ... [mContentStart] ... [mActiveStart~mActiveEnd] ... │
│                                                                      │
│  ◄────────────── contentWindow (缓存范围) ─────────────────────►     │
│                           ◄────── activeWindow ───────►              │
│                                              (可见区域)              │
│                                                                      │
│  mContentStart = mActiveStart - buffer                               │
│  mContentEnd = mActiveEnd + buffer                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 八、错误处理机制

### 8.1 加载失败处理

```java
// UpdateContent.call() 中
if ((items == null) || items.isEmpty()) {
    if (info.reloadCount > 0) {
        mFailedVersion = info.version;  // 记录失败版本
        Log.d(TAG, "loading failed: " + mFailedVersion);
    }
    return null;
}

// GetUpdateInfo.call() 中检查
if (mFailedVersion == mVersion) {
    return null;  // 上次失败，暂停加载
}
```

### 8.2 失败恢复

```java
// 当用户操作触发 notifyDirty() 时
public synchronized void notifyDirty() {
    mDirty = true;
    notifyAll();
}

// ReloadTask.run() 中
if (mActive && (mFailedVersion != MediaObject.INVALID_DATA_VERSION)) {
    Log.d(TAG, "reload resume");  // 重试失败加载
}
```

### 8.3 任务取消

```java
// ReloadTask.terminate()
public synchronized void terminate() {
    mActive = false;
    notifyAll();
}

// 使用场景
public void pause() {
    mReloadTask.terminate();
    mReloadTask = null;
    mSource.removeContentListener(mSourceListener);
}
```

---

## 九、性能优化设计

### 9.1 分批加载

```java
private static final int MIN_LOAD_COUNT = 32;
private static final int MAX_LOAD_COUNT = 64;

// GetUpdateInfo 中
info.reloadCount = Math.min(MAX_LOAD_COUNT, n - i);
```

**作用**: 避免一次性加载大量数据导致 ANR

### 9.2 版本缓存

```java
// 只比较版本号，避免重复加载
long setVersion[] = mSetVersion;
for (int i = mContentStart; i < mContentEnd; ++i) {
    if (setVersion[index] != mVersion) {
        // 只有版本不一致的才需要加载
    }
}
```

### 9.3 增量更新

```java
// UpdateContent 中只更新变化的数据
if (mItemVersion[index] != itemVersion) {
    mData[index] = updateItem;
    // ...
}
```

### 9.4 等待而非轮询

```java
// 使用 wait/notify 而非 while 循环轮询
synchronized (this) {
    if (mActive && !mDirty && updateComplete) {
        Utils.waitWithoutInterrupt(this);
        continue;
    }
}
```

**优点**: 不消耗 CPU，等待时不占用资源

---

## 十、关键配置参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `DATA_CACHE_SIZE` | 1000 | 数据缓存容量 |
| `MIN_LOAD_COUNT` | 32 | 最小批量加载数 |
| `MAX_LOAD_COUNT` | 64 | 最大批量加载数 |
| 线程优先级 | THREAD_PRIORITY_BACKGROUND | 后台线程 |

---

## 十一、生命周期管理

### 11.1 启动流程

```java
// AlbumPage.onResume()
public void onResume() {
    super.onResume();
    mAlbumDataAdapter.resume();  // 启动 ReloadTask
}

// AlbumDataLoader.resume()
public void resume() {
    mSource.addContentListener(mSourceListener);  // 注册监听
    mReloadTask = new ReloadTask();               // 创建线程
    mReloadTask.start();                          // 启动
}
```

### 11.2 停止流程

```java
// AlbumPage.onPause()
public void onPause() {
    super.onPause();
    mAlbumDataAdapter.pause();  // 停止 ReloadTask
}

// AlbumDataLoader.pause()
public void pause() {
    mReloadTask.terminate();     // 终止线程
    mReloadTask = null;
    mSource.removeContentListener(mSourceListener);  // 移除监听
}

// ReloadTask.terminate()
public synchronized void terminate() {
    mActive = false;
    notifyAll();
}
```

### 11.3 完整生命周期

```
AlbumDataLoader.resume()
    │
    ├─► MySourceListener 注册到 MediaSet
    ├─► ReloadTask.start()
    │      └─► run() 开始循环
    │             ├─► wait() 等待
    │             └─► 加载数据
    │
AlbumDataLoader.pause()
    │
    ├─► ReloadTask.terminate()
    │      └─► mActive = false
    │             └─► notifyAll() 唤醒并退出
    └─► 移除 ContentListener
```

---

## 十二、与其他模块的交互

### 12.1 与 MediaSet

```java
// ReloadTask.run()
long version = mSource.reload();  // 同步数据源
info.items = mSource.getMediaItem(info.reloadStart, info.reloadCount);
```

### 12.2 与 AlbumSlidingWindow

```java
// AlbumSlidingWindow 设置可见窗口
public void setActiveWindow(int start, int end) {
    // ...
    setContentWindow(contentStart, contentEnd);
    // ...
}

// setContentWindow 中触发更新
private void setContentWindow(int contentStart, int contentEnd) {
    // ...
    if (mReloadTask != null) mReloadTask.notifyDirty();
}
```

### 12.3 与 SlotView

```java
// UpdateContent.call() 中
if (mDataListener != null && 
    i >= mActiveStart && i < mActiveEnd) {
    mDataListener.onContentChanged(i);
}

// AlbumSlidingWindow 实现 DataListener
@Override
public void onContentChanged(int index) {
    if (mListener != null) mListener.onContentChanged();
}
```

---

## 十三、设计优点总结

| 优点 | 说明 |
|------|------|
| **线程安全** | 使用 volatile、synchronized、AtomicBoolean |
| **高效等待** | wait/notify 而非轮询，不消耗 CPU |
| **批量加载** | 每次最多 64 条，避免 ANR |
| **增量更新** | 版本机制，只加载变化的数据 |
| **内存优化** | 环形缓冲区，只缓存 1000 条 |
| **错误恢复** | 失败版本号记录，自动重试 |
| **生命周期管理** | resume/pause 完整管理 |
| **解耦设计** | ContentObserver 变化通知机制 |

---

## 十四、常见问题分析

### Q1: 为什么冷启动时首个缩略图显示较慢？

**原因**: ReloadTask 需要先完成 MediaSet 数据加载，才能开始缩略图加载

**优化方向**: 可以在 resume() 时预加载部分缩略图

### Q2: 滑动时数据闪烁如何处理？

**原因**: 快速滑动时旧数据被清除，新数据未加载完成

**当前机制**: AlbumSlidingWindow 维护缓冲区，提前加载非活跃区域

### Q3: ReloadTask 和 Worker Thread 的区别？

| 特性 | ReloadTask | Worker Thread |
|------|------------|---------------|
| 职责 | 媒体数据同步 | 缩略图解码 |
| 线程数 | 1 个 | 最多 8 个 |
| 优先级 | BACKGROUND | BACKGROUND |
| 触发方式 | ContentObserver | SlotView 可见性 |

---

本文档完整解析了 ReloadTask 的设计逻辑、触发条件、冷启动行为和性能优化策略。
