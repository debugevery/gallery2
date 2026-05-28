# Gallery2 冷启动与缓存分层处理深度分析

## 一、缓存分层架构概述

Gallery2 的缓存体系采用**四级缓存架构**，从上到下依次为：

```
┌─────────────────────────────────────────────────────────────────┐
│                     第一层：GPU纹理缓存                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  OpenGL ES 显存    │ TiledTexture / BitmapTexture        │   │
│  │  (显存/GPU内存)    │ 上传到GPU的纹理数据                  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     第二层：内存Bitmap池                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  GalleryBitmapPool (20MB)                                │   │
│  │  ├─ Square池: 正方形位图 (约6.7MB)                        │   │
│  │  ├─ Photo池: 常见照片比例4:3/3:2/16:9 (约6.7MB)           │   │
│  │  └─ Misc池: 其他比例位图 (约6.7MB)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     第三层：内存数据缓存                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  LruCache<K, V> (强引用 + 弱引用)                        │   │
│  │  ├─ MediaSet数据缓存 (1000条)                            │   │
│  │  ├─ AlbumDataLoader数据缓存                              │   │
│  │  └─ 通用对象缓存                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     第四层：磁盘缓存                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  BlobCache (ImageCacheService)                          │   │
│  │  ├─ 文件路径: /data/data/com.android.gallery/cache/     │   │
│  │  │         imgcache.idx, imgcache.0, imgcache.1         │   │
│  │  ├─ 最大条目: 5000条                                     │   │
│  │  ├─ 最大容量: 200MB                                      │   │
│  │  └─ 版本号: 7                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                     源数据：文件系统/MediaStore                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  /storage/emulated/0/DCIM/Camera/                       │   │
│  │  Content://media/external/images/media                   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、各层缓存详细分析

### 2.1 第一层：GPU纹理缓存（Texture Cache）

**核心类**：[Texture.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/Texture.java)、[TiledTexture.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/TiledTexture.java)、[BitmapTexture.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/BitmapTexture.java)

**职责**：
- 存储已经上传到GPU显存的纹理数据
- 直接被OpenGL渲染管线使用
- 避免每次绘制都上传纹理

**TiledTexture特性**：
```java
// TiledTexture.java - 将大图分割成多个瓦片
private static final int TILE_SIZE = 254;  // 每个瓦片大小（像素）
private static final int BORDER_SIZE = 1;   // 瓦片边框大小（像素）
// 实际瓦片 = 254 + 2*1 = 256像素（含边框）

// 瓦片上传队列
private static TileUploader sUploader;  // 静态上传器
```

**上传机制**：
```java
// TextureUploader.java - 管理纹理上传
// 每帧最多上传4毫秒的纹理数据
private static final long TIME_BUDGET = 4000000;  // 纳秒

public void addTexture(TiledTexture texture) {
    // 添加到上传队列
}

// 每帧调用一次
public boolean uploadOneTexture() {
    // 检查时间预算
    // 如果还有剩余时间，上传一个瓦片
    // 返回是否还有更多瓦片需要上传
}
```

**生命周期**：
```
内存Bitmap创建
    ↓
TiledTexture构造函数接收Bitmap
    ↓
在GL渲染线程调用upload()
    ↓
glTexImage2D() 上传到GPU显存
    ↓
Bitmap可以recycle()释放内存
    ↓
只保留GPU显存中的纹理数据
```

### 2.2 第二层：GalleryBitmapPool（位图复用池）

**核心类**：[GalleryBitmapPool.java](file:///d:/code/gallery2/Gallery2/src/com/android/photos/data/GalleryBitmapPool.java)、[SparseArrayBitmapPool.java](file:///d:/code/gallery2/Gallery2/src/com/android/photos/data/SparseArrayBitmapPool.java)

**容量配置**：
```java
// GalleryBitmapPool.java
private static final int CAPACITY_BYTES = 20971520;  // 20MB

// 分为三个子池
private static final int POOL_INDEX_SQUARE = 0;   // 正方形
private static final int POOL_INDEX_PHOTO = 1;     // 常见照片比例
private static final int POOL_INDEX_MISC = 2;      // 其他比例

// 每个子池约 20MB / 3 ≈ 6.7MB
mPools[POOL_INDEX_SQUARE] = new SparseArrayBitmapPool(capacityBytes / 3, ...);
mPools[POOL_INDEX_PHOTO] = new SparseArrayBitmapPool(capacityBytes / 3, ...);
mPools[POOL_INDEX_MISC] = new SparseArrayBitmapPool(capacityBytes / 3, ...);
```

**按尺寸分类**：
```java
// 根据宽高比分类
private static final Point[] COMMON_PHOTO_ASPECT_RATIOS =
    { new Point(4, 3), new Point(3, 2), new Point(16, 9) };

private int getPoolIndexForDimensions(int width, int height) {
    if (width == height) {
        return POOL_INDEX_SQUARE;  // 正方形 → Square池
    }
    // 检查是否是常见照片比例
    for (Point ar : COMMON_PHOTO_ASPECT_RATIOS) {
        if (min * ar.x == max * ar.y) {
            return POOL_INDEX_PHOTO;  // 4:3/3:2/16:9 → Photo池
        }
    }
    return POOL_INDEX_MISC;  // 其他 → Misc池
}
```

**FIFO淘汰策略**：
```java
// SparseArrayBitmapPool.java - 双向链表实现FIFO
private Node mPoolNodesHead = null;  // 头部（最新）
private Node mPoolNodesTail = null;  // 尾部（最旧）

// 添加新节点到头部
newNode.nextInPool = mPoolNodesHead;
mPoolNodesHead = newNode;

// 容量满时从尾部淘汰
while (mPoolNodesTail != null && mSizeBytes > targetSize) {
    unlinkAndRecycleNode(mPoolNodesTail, true);  // recycle Bitmap
}
```

**复用流程**：
```java
// DecodeUtils.decodeUsingPool() 中使用
options.inBitmap = GalleryBitmapPool.getInstance().get(width, height);
Bitmap bitmap = BitmapFactory.decodeFileDescriptor(fd, null, options);
if (options.inBitmap != null && options.inBitmap != bitmap) {
    // 解码到了不同的Bitmap（可能是新分配的），把原来复用的放回池中
    GalleryBitmapPool.getInstance().put(options.inBitmap);
}
```

### 2.3 第三层：LruCache（内存数据缓存）

**核心类**：[LruCache.java](file:///d:/code/gallery2/Gallery2/gallerycommon/src/com/android/gallery3d/common/LruCache.java)

**双引用设计**：
```java
// LruCache.java - 同时维护强引用和弱引用
private final HashMap<K, V> mLruMap;          // 强引用（LRU）
private final HashMap<K, Entry<K, V>> mWeakMap;  // 弱引用
private ReferenceQueue<V> mQueue;              // 引用队列

// LinkedHashMap的accessOrder=true，按访问顺序排序
mLruMap = new LinkedHashMap<K, V>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;  // 超过容量时移除最老的
    }
};
```

**Entry弱引用**：
```java
private static class Entry<K, V> extends WeakReference<V> {
    K mKey;
    public Entry(K key, V value, ReferenceQueue<V> queue) {
        super(value, queue);
        mKey = key;
    }
}
```

**获取逻辑**：
```java
public synchronized V get(K key) {
    cleanUpWeakMap();  // 先清理已被GC的弱引用
    V value = mLruMap.get(key);
    if (value != null) return value;  // 优先返回强引用
    // 强引用没有，返回弱引用（可能被GC了）
    Entry<K, V> entry = mWeakMap.get(key);
    return entry == null ? null : entry.get();
}
```

**存储逻辑**：
```java
public synchronized V put(K key, V value) {
    cleanUpWeakMap();
    mLruMap.put(key, value);  // 添加到LRU头部
    // 同时添加到弱引用map（即使被LRU淘汰，弱引用可能还能访问）
    mWeakMap.put(key, new Entry<K, V>(key, value, mQueue));
}
```

**在宫格页的使用**：
```java
// AlbumDataLoader.java - 媒体项缓存
private static final int DATA_CACHE_SIZE = 1000;
// MediaItem[] mData = new MediaItem[DATA_CACHE_SIZE];
// 存储最近访问的1000个媒体项
```

### 2.4 第四层：ImageCacheService（磁盘缓存）

**核心类**：[ImageCacheService.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/data/ImageCacheService.java)、[BlobCache.java](file:///d:/code/gallery2/Gallery2/gallerycommon/src/com/android/gallery3d/common/BlobCache.java)

**配置参数**：
```java
// ImageCacheService.java
private static final String IMAGE_CACHE_FILE = "imgcache";
private static final int IMAGE_CACHE_MAX_ENTRIES = 5000;      // 最多5000条
private static final int IMAGE_CACHE_MAX_BYTES = 200 * 1024 * 1024;  // 200MB
private static final int IMAGE_CACHE_VERSION = 7;             // 版本号（变化时清空）

// BlobCache文件
// imgcache.idx  - 索引文件
// imgcache.0    - 数据文件区域0
// imgcache.1    - 数据文件区域1
```

**缓存键生成**：
```java
private static byte[] makeKey(Path path, long timeModified, int type) {
    // 键格式: "path + timeModified + type"
    return GalleryUtils.getBytes(path.toString() + "+" + timeModified + "+" + type);
}

// CRC64哈希
long cacheKey = Utils.crc64Long(key);
```

**双区域设计**：
```java
// BlobCache.java - 双区域循环使用
// 区域0和区域1交替使用
// 写入流程：
// 1. 向活跃区域追加数据
// 2. 当活跃区域满或负载因子>0.5时
// 3. 交换活跃/非活跃区域
// 4. 清空新活跃区域（从0开始）
// 5. 重建索引
```

**查找流程**：
```java
public boolean getImageData(Path path, long timeModified, int type, BytesBuffer buffer) {
    byte[] key = makeKey(path, timeModified, type);
    long cacheKey = Utils.crc64Long(key);
    LookupRequest request = new LookupRequest();
    request.key = cacheKey;
    request.buffer = buffer.data;
    synchronized (mCache) {
        if (!mCache.lookup(request)) return false;  // 未命中
    }
    // 验证键匹配
    if (isSameKey(key, request.buffer)) {
        buffer.data = request.buffer;
        buffer.offset = key.length;
        buffer.length = request.length - buffer.offset;
        return true;  // 命中
    }
    return false;
}
```

---

## 三、冷启动完整流程分析

### 3.1 应用启动时序

```
用户点击图标
    ↓
[0ms] ActivityThread.main()
    ↓
[5ms] Application.onCreate()
    ├─ GalleryAppImpl.onCreate()
    │   ├─ GalleryUtils.initialize()      // 初始化工具类
    │   ├─ WidgetUtils.initialize()       // 小部件初始化
    │   └─ UsageStatistics.initialize()  // 统计初始化
    ↓
[10ms] Activity.onCreate()
    └─ GalleryActivity.onCreate()
        ├─ setContentView()
        │   └─ LayoutInflater.inflate(R.layout.main)
        │       └─ GLRootView创建
        ├─ initializeByIntent()           // 根据Intent初始化
        └─ StateManager初始化
            └─ createGalleryAppAndDataAdapter()
                ├─ getDataManager()       // 初始化数据管理器
                │   ├─ new DataManager()
                │   └─ initializeSourceMap()
                │       ├─ LocalSource注册
                │       ├─ ComboSource注册
                │       └─ FilterSource注册
                └─ getImageCacheService() // 初始化缓存服务
                    └─ new ImageCacheService()
                        └─ CacheManager.getCache()
                            └─ BlobCache构造函数
                                ├─ 如果没有缓存文件 → 创建新缓存
                                └─ 如果有缓存文件 → loadIndex()
    ↓
[15ms] Activity.onResume()
    └─ GLRootView.onResume()
        └─ startRendering()  // 启动GL渲染循环
    ↓
[20ms] 首页State激活
    └─ 根据Intent决定进入哪个页面
        ├─ Intent.ACTION_MAIN → AlbumSetPage（所有相册）
        └─ 指定相册路径 → AlbumPage（具体相册）
```

### 3.2 缓存服务初始化

```java
// GalleryAppImpl.java
@Override
public ImageCacheService getImageCacheService() {
    synchronized (mLock) {
        if (mImageCacheService == null) {
            mImageCacheService = new ImageCacheService(getAndroidContext());
            // 内部会调用 CacheManager.getCache()
        }
        return mImageCacheService;
    }
}

// ImageCacheService构造函数
public ImageCacheService(Context context) {
    mCache = CacheManager.getCache(context, IMAGE_CACHE_FILE,
            IMAGE_CACHE_MAX_ENTRIES, IMAGE_CACHE_MAX_BYTES,
            IMAGE_CACHE_VERSION);
}
```

```java
// CacheManager.java - 缓存管理
private static final String CACHE_DIR = "image_cache";

public static BlobCache getCache(Context context, String filename,
        int maxEntries, int maxBytes, int version) {
    File cacheDir = new File(context.getCacheDir(), CACHE_DIR);
    String path = new File(cacheDir, filename).getAbsolutePath();
    
    try {
        return new BlobCache(path, maxEntries, maxBytes, false, version);
    } catch (IOException e) {
        // 缓存文件损坏，删除重建
        BlobCache.deleteFiles(path);
        try {
            return new BlobCache(path, maxEntries, maxBytes, true, version);
        } catch (IOException e2) {
            return null;
        }
    }
}
```

### 3.3 BlobCache初始化细节

```java
// BlobCache.java构造函数
public BlobCache(String path, int maxEntries, int maxBytes, boolean reset, int version) {
    mIndexFile = new RandomAccessFile(path + ".idx", "rw");
    mDataFile0 = new RandomAccessFile(path + ".0", "rw");
    mDataFile1 = new RandomAccessFile(path + ".1", "rw");
    mVersion = version;

    if (!reset && loadIndex()) {
        return;  // 成功加载已有缓存
    }
    
    resetCache(maxEntries, maxBytes);  // 重置/创建新缓存
    
    if (!loadIndex()) {
        closeAll();
        throw new IOException("unable to load index");
    }
}
```

**索引加载流程**：
```java
private boolean loadIndex() {
    mIndexFile.seek(0);
    mDataFile0.seek(0);
    mDataFile1.seek(0);

    byte[] buf = mIndexHeader;
    if (mIndexFile.read(buf) != INDEX_HEADER_SIZE) {
        return false;  // 无法读取头部
    }

    // 验证魔数
    int magic = readInt(buf, IH_MAGIC);
    if (magic != MAGIC_INDEX_FILE) {
        return false;
    }

    // 验证校验和
    int checksum = readInt(buf, IH_CHECKSUM);
    if (checksum != getChecksum(buf, 0, IH_CHECKSUM)) {
        return false;
    }

    // 读取配置
    mMaxEntries = readInt(buf, IH_MAX_ENTRIES);
    mMaxBytes = readInt(buf, IH_MAX_BYTES);
    mActiveRegion = readInt(buf, IH_ACTIVE_REGION);
    mActiveEntries = readInt(buf, IH_ACTIVE_ENTRIES);
    mActiveBytes = readInt(buf, IH_ACTIVE_BYTES);

    // 内存映射索引文件（加速访问）
    mIndexChannel = mIndexFile.getChannel();
    mIndexBuffer = mIndexChannel.map(FileChannel.MapMode.READ_WRITE, 0, 
            INDEX_HEADER_SIZE + 2 * mMaxEntries * 12);

    // 设置活跃/非活跃数据文件
    mActiveDataFile = (mActiveRegion == 0) ? mDataFile0 : mDataFile1;
    mInactiveDataFile = (mActiveRegion == 0) ? mDataFile1 : mDataFile0;

    return true;
}
```

### 3.4 GalleryBitmapPool初始化

```java
// GalleryBitmapPool.java - 静态单例
private static GalleryBitmapPool sInstance = new GalleryBitmapPool(CAPACITY_BYTES);

public static GalleryBitmapPool getInstance() {
    return sInstance;
}

// 构造函数
private GalleryBitmapPool(int capacityBytes) {
    mPools = new SparseArrayBitmapPool[3];
    // 每个子池分配1/3容量
    mPools[POOL_INDEX_SQUARE] = new SparseArrayBitmapPool(capacityBytes / 3, mSharedNodePool);
    mPools[POOL_INDEX_PHOTO] = new SparseArrayBitmapPool(capacityBytes / 3, mSharedNodePool);
    mPools[POOL_INDEX_MISC] = new SparseArrayBitmapPool(capacityBytes / 3, mSharedNodePool);
    mCapacityBytes = capacityBytes;
}
```

**启动时状态**：
- 初始为空（20MB容量全部可用）
- 等待解码时申请可复用Bitmap
- 回收时存入池中

### 3.5 ThreadPool初始化

```java
// GalleryAppImpl.getThreadPool()
@Override
public synchronized ThreadPool getThreadPool() {
    if (mThreadPool == null) {
        mThreadPool = new ThreadPool();
    }
    return mThreadPool;
}

// ThreadPool.java
private static final int CORE_POOL_SIZE = 4;      // 核心线程数
private static final int MAX_POOL_SIZE = 8;       // 最大线程数
private static final int KEEP_ALIVE_TIME = 10;    // 空闲存活时间（秒）

public ThreadPool() {
    // 使用PriorityThreadFactory创建线程
    // 线程优先级: THREAD_PRIORITY_BACKGROUND
}
```

---

## 四、宫格页数据加载与缓存流程

### 4.1 相册页面创建

```java
// AlbumPage.java - onCreate()
public void onCreate(Bundle data, Bundle restoreState) {
    // 1. 初始化视图
    initializeViews();
    // 2. 初始化数据
    initializeData(data);
}

private void initializeViews() {
    // 创建SlotView（宫格容器）
    mSlotView = new SlotView(mActivity, config.slotViewSpec);
    
    // 创建渲染器
    mAlbumView = new AlbumSlotRenderer(mActivity, ...);
    mSlotView.setSlotRenderer(mAlbumView);
    
    // 创建数据加载器
    mAlbumDataAdapter = new AlbumDataLoader(mActivity, mMediaSet);
    mAlbumView.setModel(mAlbumDataAdapter);
    
    // 设置可见性变化监听
    mSlotView.setListener(new SlotViewListener());
}

private void initializeData(Bundle data) {
    String mediaPath = data.getString(KEY_MEDIA_PATH);
    mMediaSetPath = Path.fromString(mediaPath);
    mMediaSet = mActivity.getDataManager().getMediaSet(mMediaSetPath);
    mAlbumDataAdapter = new AlbumDataLoader(mActivity, mMediaSet);
}
```

### 4.2 数据加载器启动

```java
// AlbumPage.onResume()
public void onResume() {
    super.onResume();
    mAlbumDataAdapter.resume();
    mAlbumView.resume();
}

// AlbumDataLoader.resume()
public void resume() {
    mSource.addContentListener(this);  // 注册内容变化监听
    mReloadTask.start();              // 启动后台加载线程
}

public void pause() {
    mSource.removeContentListener(this);
    mReloadTask.finish();             // 停止后台加载
}
```

### 4.3 ReloadTask后台加载

```java
// AlbumDataLoader内部类
private class ReloadTask extends Thread {
    public void run() {
        while (mActive) {
            // 1. 同步源数据
            int syncResult = mSource.reload();
            
            // 2. 获取更新信息（在主线程执行）
            GetUpdateInfo info = executeAndWait(new GetUpdateInfo());
            
            // 3. 加载媒体项
            info.items = mSource.getMediaItem(info.startIndex, info.count);
            
            // 4. 更新内容（在主线程执行）
            executeAndWait(new UpdateContent(info));
            
            // 5. 等待变化通知
            synchronized (mLock) {
                while (mActive && !mNeedReload) {
                    try {
                        mLock.wait();
                    } catch (InterruptedException e) {}
                }
                mNeedReload = false;
            }
        }
    }
}
```

### 4.4 AlbumSlidingWindow滑动窗口

```java
// AlbumSlidingWindow构造函数
public AlbumSlidingWindow(AbstractGalleryActivity activity,
        AlbumDataLoader source, int cacheSize) {
    source.setDataListener(this);  // 设置数据监听
    mSource = source;
    mData = new AlbumEntry[cacheSize];  // CACHE_SIZE = 96
    mSize = source.size();

    // 创建Handler（绑定到GL渲染线程）
    mHandler = new SynchronizedHandler(activity.getGLRoot()) {
        @Override
        public void handleMessage(Message message) {
            ((ThumbnailLoader) message.obj).updateEntry();
        }
    };

    // 创建线程限制器（最多2个并发）
    mThreadPool = new JobLimiter(activity.getThreadPool(), JOB_LIMIT);
    
    // 创建纹理上传器
    mTileUploader = new TiledTexture.Uploader(activity.getGLRoot());
}
```

### 4.5 窗口设置与内容准备

```java
// SlotView滚动时调用
public void setActiveWindow(int start, int end) {
    mActiveStart = start;
    mActiveEnd = end;

    // 计算内容窗口（居中，包含更多缓冲）
    int contentStart = Utils.clamp((start + end) / 2 - data.length / 2,
            0, Math.max(0, mSize - data.length));
    int contentEnd = Math.min(contentStart + data.length, mSize);
    
    setContentWindow(contentStart, contentEnd);  // 设置内容缓存窗口
    
    updateTextureUploadQueue();  // 更新纹理上传队列
    
    if (mIsActive) updateAllImageRequests();  // 更新图像请求
}
```

### 4.6 内容窗口设置

```java
private void setContentWindow(int contentStart, int contentEnd) {
    if (contentStart == mContentStart && contentEnd == mContentEnd) return;

    if (!mIsActive) {
        // 非活跃状态，只更新范围
        mContentStart = contentStart;
        mContentEnd = contentEnd;
        mSource.setActiveWindow(contentStart, contentEnd);
        return;
    }

    if (contentStart >= mContentEnd || mContentStart >= contentEnd) {
        // 完全不重叠，完全替换
        for (int i = mContentStart; i < mContentEnd; ++i) {
            freeSlotContent(i);  // 释放旧内容
        }
        mSource.setActiveWindow(contentStart, contentEnd);
        for (int i = contentStart; i < contentEnd; ++i) {
            prepareSlotContent(i);  // 准备新内容
        }
    } else {
        // 部分重叠，只更新差异部分
        // ... 省略部分逻辑
    }

    mContentStart = contentStart;
    mContentEnd = contentEnd;
}
```

### 4.7 槽位内容准备

```java
private void prepareSlotContent(int slotIndex) {
    AlbumEntry entry = new AlbumEntry();
    MediaItem item = mSource.get(slotIndex);  // 从DataLoader获取
    entry.item = item;
    entry.mediaType = (item == null) 
            ? MediaItem.MEDIA_TYPE_UNKNOWN 
            : entry.item.getMediaType();
    entry.path = (item == null) ? null : item.getPath();
    entry.rotation = (item == null) ? 0 : item.getRotation();
    
    // 创建缩略图加载器
    entry.contentLoader = new ThumbnailLoader(slotIndex, entry.item);
    
    mData[slotIndex % mData.length] = entry;
}
```

---

## 五、缩略图加载完整链路

### 5.1 发起请求

```java
// AlbumSlidingWindow.requestSlotImage()
private boolean requestSlotImage(int slotIndex) {
    if (slotIndex < mContentStart || slotIndex >= mContentEnd) return false;
    AlbumEntry entry = mData[slotIndex % mData.length];
    if (entry.content != null || entry.item == null) return false;

    // 启动缩略图加载
    entry.contentLoader.startLoad();
    return entry.contentLoader.isRequestInProgress();
}

// ThumbnailLoader.startLoad()
public synchronized void startLoad() {
    if (mState == STATE_INIT) {
        mState = STATE_REQUESTED;
        if (mTask == null) {
            mTask = submitBitmapTask(this);  // 提交到线程池
        }
    }
}
```

### 5.2 线程池调度

```java
// ThumbnailLoader.submitBitmapTask()
@Override
protected Future<Bitmap> submitBitmapTask(FutureListener<Bitmap> l) {
    return mThreadPool.submit(
            mItem.requestImage(MediaItem.TYPE_MICROTHUMBNAIL), this);
}
```

### 5.3 MediaItem请求处理

```java
// LocalImage.requestImage()
public Future<Bitmap> requestImage(int type) {
    return mContext.getThreadPool().submit(
            new Job<Bitmap>() {
                public Bitmap run(JobContext jc) {
                    return decodeThumbnail(jc, type);
                }
            });
}

private Bitmap decodeThumbnail(JobContext jc, int type) {
    // 1. 尝试从磁盘缓存读取
    BytesBuffer buffer = BytesBufferPool.getBuffer();
    boolean found = mContext.getImageCacheService().getImageData(
            getPath(), mDateModified, type, buffer);
    
    if (found) {
        // 命中磁盘缓存，直接解码
        Bitmap bitmap = DecodeUtils.decode(jc, buffer.data, 
                buffer.offset, buffer.length, null);
        BytesBufferPool.recycle(buffer);
        return bitmap;
    }
    
    // 2. 未命中，从文件解码
    Bitmap bitmap = DecodeUtils.decodeThumbnail(jc, getFilePath(), 
            null, sMicroThumbnailTargetSize, type);
    
    // 3. 存入磁盘缓存
    if (bitmap != null) {
        mContext.getImageCacheService().putImageData(
                getPath(), mDateModified, type, bitmapToByteArray(bitmap));
    }
    
    return bitmap;
}
```

### 5.4 四级缓存查找

```
LocalImage.decodeThumbnail()
    │
    ├─► 第一层：ImageCacheService（磁盘缓存）
    │       │
    │       └─► BlobCache.lookup(cacheKey)
    │           ├─ 命中 → 返回压缩数据 → DecodeUtils.decode()
    │           └─ 未命中 → 继续
    │
    ├─► 第二层：从文件解码
    │       │
    │       └─► DecodeUtils.decodeThumbnail()
    │           │
    │           ├─► GalleryBitmapPool.get(width, height)  // 解码前尝试复用
    │           │
    │           └─► BitmapFactory.decodeFileDescriptor()
    │               │
    │               └─► 存入ImageCacheService.putImageData()
    │
    └─► 返回 Bitmap
```

### 5.5 解码优化

```java
// DecodeUtils.decodeThumbnail()
public static Bitmap decodeThumbnail(JobContext jc, String filePath, 
        Options options, int targetSize, int type) {
    FileInputStream fis = null;
    try {
        fis = new FileInputStream(filePath);
        FileDescriptor fd = fis.getFD();
        return decodeThumbnail(jc, fd, options, targetSize, type);
    } finally {
        Utils.closeSilently(fis);
    }
}

public static Bitmap decodeThumbnail(JobContext jc, FileDescriptor fd, 
        Options options, int targetSize, int type) {
    if (options == null) options = new Options();
    jc.setCancelListener(new DecodeCanceller(options));

    // 第一步：只解码尺寸
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeFileDescriptor(fd, null, options);
    if (jc.isCancelled()) return null;

    int w = options.outWidth;
    int h = options.outHeight;

    // 第二步：计算采样率
    if (type == MediaItem.TYPE_MICROTHUMBNAIL) {
        // 中心裁剪模式，较短边 >= targetSize
        float scale = (float) targetSize / Math.min(w, h);
        options.inSampleSize = BitmapUtils.computeSampleSizeLarger(scale);
        
        // 防止超大图片OOM
        final int MAX_PIXEL_COUNT = 640000;  // 400 x 1600
        if ((w / options.inSampleSize) * (h / options.inSampleSize) > MAX_PIXEL_COUNT) {
            options.inSampleSize = BitmapUtils.computeSampleSize(
                    (float) Math.sqrt((double) MAX_PIXEL_COUNT / (w * h)));
        }
    } else {
        // 普通模式，较长边 >= targetSize
        float scale = (float) targetSize / Math.max(w, h);
        options.inSampleSize = BitmapUtils.computeSampleSizeLarger(scale);
    }

    // 第三步：实际解码
    options.inJustDecodeBounds = false;
    setOptionsMutable(options);  // 设置inMutable=true以支持复用

    Bitmap result = BitmapFactory.decodeFileDescriptor(fd, null, options);
    if (result == null) return null;

    // 第四步：必要时缩放
    float scale = (float) targetSize / (type == MediaItem.TYPE_MICROTHUMBNAIL
            ? Math.min(result.getWidth(), result.getHeight())
            : Math.max(result.getWidth(), result.getHeight()));

    if (scale <= 0.5) {
        result = BitmapUtils.resizeBitmapByScale(result, scale, true);
    }
    
    // 第五步：确保GL兼容格式
    return ensureGLCompatibleBitmap(result);
}
```

### 5.6 Bitmap复用

```java
// DecodeUtils.decodeUsingPool()
@TargetApi(Build.VERSION_CODES.HONEYCOMB)
public static Bitmap decodeUsingPool(JobContext jc,
        FileDescriptor fileDescriptor, Options options) {
    if (options == null) options = new Options();
    if (options.inSampleSize < 1) options.inSampleSize = 1;
    options.inPreferredConfig = Bitmap.Config.ARGB_8888;
    
    // 尝试从Bitmap池复用
    options.inBitmap = (options.inSampleSize == 1)
            ? findCachedBitmap(jc, fileDescriptor, options) : null;
    
    try {
        Bitmap bitmap = DecodeUtils.decode(jc, fileDescriptor, options);
        if (options.inBitmap != null && options.inBitmap != bitmap) {
            // 复用的Bitmap未被使用，存入池中
            GalleryBitmapPool.getInstance().put(options.inBitmap);
            options.inBitmap = null;
        }
        return bitmap;
    } catch (IllegalArgumentException e) {
        // 复用失败，尝试不用复用
        if (options.inBitmap == null) throw e;
        GalleryBitmapPool.getInstance().put(options.inBitmap);
        options.inBitmap = null;
        return decode(jc, fileDescriptor, options);
    }
}

private static Bitmap findCachedBitmap(JobContext jc, FileDescriptor fileDescriptor,
        Options options) {
    decodeBounds(jc, fileDescriptor, options);
    return GalleryBitmapPool.getInstance().get(options.outWidth, options.outHeight);
}
```

### 5.7 加载完成回调

```java
// BitmapLoader.onFutureDone()
@Override
public void onFutureDone(Future<Bitmap> future) {
    synchronized (this) {
        mTask = null;
        mBitmap = future.get();
        
        if (mState == STATE_RECYCLED) {
            // 已被回收，Bitmap放回池中
            if (mBitmap != null) {
                GalleryBitmapPool.getInstance().put(mBitmap);
                mBitmap = null;
            }
            return;
        }
        
        if (future.isCancelled() && mBitmap == null) {
            // 被取消但没有Bitmap，重新提交
            if (mState == STATE_REQUESTED) {
                mTask = submitBitmapTask(this);
            }
            return;
        }
        
        mState = mBitmap == null ? STATE_ERROR : STATE_LOADED;
    }
    onLoadComplete(mBitmap);  // 调用子类回调
}
```

### 5.8 GL线程更新

```java
// ThumbnailLoader.onLoadComplete()
@Override
protected void onLoadComplete(Bitmap bitmap) {
    // 发送消息到GL线程
    mHandler.obtainMessage(MSG_UPDATE_ENTRY, this).sendToTarget();
}

// AlbumSlidingWindow.SynchronizedHandler
public void handleMessage(Message message) {
    Utils.assertTrue(message.what == MSG_UPDATE_ENTRY);
    ((ThumbnailLoader) message.obj).updateEntry();
}

// ThumbnailLoader.updateEntry()
public void updateEntry() {
    Bitmap bitmap = getBitmap();
    if (bitmap == null) return;  // 错误或已被回收

    AlbumEntry entry = mData[mSlotIndex % mData.length];
    
    // 创建TiledTexture
    entry.bitmapTexture = new TiledTexture(bitmap);
    entry.content = entry.bitmapTexture;

    if (isActiveSlot(mSlotIndex)) {
        // 活跃槽位，添加到上传队列
        mTileUploader.addTexture(entry.bitmapTexture);
        --mActiveRequestCount;
        
        // 所有活跃请求完成，开始加载非活跃槽位
        if (mActiveRequestCount == 0) {
            requestNonactiveImages();
        }
        
        // 通知内容变化
        if (mListener != null) mListener.onContentChanged();
    } else {
        // 非活跃槽位，也添加到上传队列（预加载）
        mTileUploader.addTexture(entry.bitmapTexture);
    }
}
```

---

## 六、纹理上传与渲染

### 6.1 纹理上传队列

```java
// TiledTexture.Uploader
private ArrayList<TiledTexture> mTextures = new ArrayList<>();
private int mCurrentIndex = 0;

public void addTexture(TiledTexture texture) {
    if (!mTextures.contains(texture)) {
        mTextures.add(texture);
    }
}

public boolean uploadTextures(long timeBudget) {
    // 每帧最多上传 timeBudget 纳秒
    long startTime = System.nanoTime();
    int uploaded = 0;
    
    while (mCurrentIndex < mTextures.size()) {
        TiledTexture t = mTextures.get(mCurrentIndex);
        if (!t.isUploaded()) {
            t.upload();  // 上传到GPU
            uploaded++;
        }
        mCurrentIndex++;
        
        // 检查时间预算
        if (System.nanoTime() - startTime >= timeBudget) {
            break;
        }
    }
    
    return mCurrentIndex < mTextures.size();  // 是否还有更多
}

public void clear() {
    mTextures.clear();
    mCurrentIndex = 0;
}
```

### 6.2 TiledTexture上传

```java
// TiledTexture.java
public synchronized void upload() {
    if (mUploaded) return;
    
    int width = mBitmap.getWidth();
    int height = mBitmap.getHeight();
    
    // 计算瓦片数量
    int tileWidth = (width + TILE_SIZE - 1) / TILE_SIZE;
    int tileHeight = (height + TILE_SIZE - 1) / TILE_SIZE;
    mTiles = new Tile[tileWidth * tileHeight];
    
    // 生成纹理ID
    int[] ids = new int[1];
    GLES20.glGenTextures(1, ids, 0);
    mTextureId = ids[0];
    
    // 绑定纹理
    GLES20.glBindTexture(GLES11Compat.GL_TEXTURE_2D, mTextureId);
    GLES20.glTexParameteri(GLES11Compat.GL_TEXTURE_2D, 
            GLES11Compat.GL_TEXTURE_MIN_FILTER, GLES11Compat.GL_LINEAR_MIPMAP_LINEAR);
    GLES20.glTexParameteri(GLES11Compat.GL_TEXTURE_2D, 
            GLES11Compat.GL_TEXTURE_MAG_FILTER, GLES11Compat.GL_LINEAR);
    GLES20.glTexParameteri(GLES11Compat.GL_TEXTURE_2D, 
            GLES11Compat.GL_TEXTURE_WRAP_S, GLES11Compat.GL_CLAMP_TO_EDGE);
    GLES20.glTexParameteri(GLES11Compat.GL_TEXTURE_2D, 
            GLES11Compat.GL_TEXTURE_WRAP_T, GLES11Compat.GL_CLAMP_TO_EDGE);
    
    // 上传Bitmap数据
    GLUtils.texImage2D(GLES11Compat.GL_TEXTURE_2D, 0, mBitmap, 0);
    
    // 生成MipMap
    GLES20.glGenerateMipmap(GLES11Compat.GL_TEXTURE_2D);
    
    // 释放Bitmap内存（GPU已有数据）
    mBitmap.recycle();
    mBitmap = null;
    
    mUploaded = true;
}
```

### 6.3 宫格渲染

```java
// AlbumSlotRenderer.renderSlot()
public void renderSlot(GLCanvas canvas, int index, int pass, int width, int height) {
    AlbumEntry entry = mData.get(index);
    
    // 检查纹理是否就绪
    Texture content = checkTexture(entry.content);
    
    if (content == null) {
        // 未加载完成，显示占位符
        canvas.drawTexture(mWaitLoadingTexture, 0, 0, width, height);
        return;
    }
    
    // 绘制内容
    drawContent(canvas, content, width, height, entry.rotation);
    
    // 绘制视频/全景图标
    if (entry.mediaType == MediaItem.MEDIA_TYPE_VIDEO) {
        drawVideoOverlay(canvas, width, height);
    }
    
    if (entry.isPanorama) {
        drawPanoramaIcon(canvas, width, height);
    }
    
    // 绘制选中/按下状态
    renderOverlay(canvas, index, width, height);
}

private void drawContent(GLCanvas canvas, Texture content, 
        int width, int height, int rotation) {
    if (rotation != 0) {
        canvas.save();
        canvas.rotate(rotation, width / 2, height / 2);
    }
    
    if (content instanceof TiledTexture) {
        TiledTexture tt = (TiledTexture) content;
        tt.draw(canvas, 0, 0, width, height);
    } else {
        canvas.drawTexture(content, 0, 0, width, height);
    }
    
    if (rotation != 0) {
        canvas.restore();
    }
}
```

---

## 七、暂停与恢复

### 7.1 页面暂停

```java
// AlbumPage.onPause()
public void onPause() {
    super.onPause();
    mAlbumDataAdapter.pause();
    mAlbumView.pause();
}

// AlbumSlidingWindow.pause()
public void pause() {
    mIsActive = false;
    mTileUploader.clear();  // 清空上传队列
    TiledTexture.freeResources();  // 释放GPU资源
    
    // 释放所有槽位内容
    for (int i = mContentStart; i < mContentEnd; ++i) {
        freeSlotContent(i);
    }
}
```

### 7.2 页面恢复

```java
// AlbumPage.onResume()
public void onResume() {
    super.onResume();
    mAlbumDataAdapter.resume();
    mAlbumView.resume();
}

// AlbumSlidingWindow.resume()
public void resume() {
    mIsActive = true;
    TiledTexture.prepareResources();  // 准备GPU资源
    
    // 重新准备所有槽位内容
    for (int i = mContentStart; i < mContentEnd; ++i) {
        prepareSlotContent(i);
    }
    
    updateAllImageRequests();  // 重新发起图像请求
}
```

### 7.3 TiledTexture资源管理

```java
// TiledTexture.java
private static boolean sResourcesPrepared = false;

public static void prepareResources() {
    if (!sResourcesPrepared) {
        // 预热GPU资源
        sResourcesPrepared = true;
    }
}

public static void freeResources() {
    if (sResourcesPrepared) {
        // 清理GPU资源
        sResourcesPrepared = false;
    }
}
```

---

## 八、内存清理机制

### 8.1 LRU淘汰

```java
// LruCache.java
private final HashMap<K, V> mLruMap = new LinkedHashMap<K, V>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;  // 超过容量时移除最老的
    }
};

// 当put新条目时，如果超过容量，会自动removeEldestEntry
// 最老的是最少访问的（accessOrder=true）
```

### 8.2 Bitmap池淘汰

```java
// SparseArrayBitmapPool.java
private void freeUpCapacity(int bytesNeeded) {
    int targetSize = mCapacityBytes - bytesNeeded;
    
    // 从尾部（FIFO，最老的）开始淘汰
    while (mPoolNodesTail != null && mSizeBytes > targetSize) {
        unlinkAndRecycleNode(mPoolNodesTail, true);  // recycle Bitmap
    }
}

private void unlinkAndRecycleNode(Node n, boolean recycleBitmap) {
    // 从链表中移除
    // ...
    
    // 回收Bitmap
    mSizeBytes -= n.bitmap.getByteCount();
    if (recycleBitmap) n.bitmap.recycle();
    
    // 释放节点回节点池
    mNodePool.release(n);
}
```

### 8.3 BlobCache淘汰

```java
// BlobCache.java - 双区域FIFO
// 当活跃区域满或负载因子>0.5时：
// 1. 交换活跃/非活跃区域
// 2. 清空新活跃区域
// 3. 重建索引
// 4. 旧区域数据自然废弃

private void setActiveRegion(int region) {
    mActiveRegion = region;
    mActiveDataFile = (region == 0) ? mDataFile0 : mDataFile1;
    mInactiveDataFile = (region == 0) ? mDataFile1 : mDataFile0;
    mActiveEntries = 0;
    mActiveBytes = DATA_HEADER_SIZE;  // 留4字节给魔数
    writeIndexHeader();
}
```

---

## 九、冷启动性能优化总结

### 9.1 缓存预热

| 阶段 | 优化措施 |
|------|---------|
| Application.onCreate() | 初始化ImageCacheService（加载索引） |
| Activity.onCreate() | 预创建线程池 |
| 第一帧渲染前 | 预加载可见区域数据 |

### 9.2 延迟加载

| 策略 | 说明 |
|------|------|
| 滑动窗口 | 只加载可见区域 + 缓冲区域 |
| 纹理上传 | 每帧最多4ms上传时间 |
| 非活跃加载 | 活跃请求完成后才加载非活跃 |

### 9.3 内存管理

| 策略 | 说明 |
|------|------|
| Bitmap复用 | GalleryBitmapPool复用20MB |
| 弱引用缓存 | LruCache双重引用 |
| GPU资源释放 | pause时释放Texture资源 |

### 9.4 时间线估算

```
[0ms]     Application.onCreate()
[5ms]     初始化GalleryAppImpl组件
[10ms]    初始化ImageCacheService（加载BlobCache索引）
[15ms]    Activity.onCreate()完成
[20ms]    GL渲染循环启动
[25ms]    AlbumDataLoader.resume()启动
[30ms]    ReloadTask开始加载数据
[50ms]    MediaSet数据加载完成
[60ms]    SlotView布局完成，开始渲染
[65ms]    发起缩略图请求
[80ms]    首个缩略图解码完成
[100ms]   首个缩略图显示到屏幕
```

---

## 十、关键配置参数汇总

| 参数 | 值 | 说明 |
|------|-----|------|
| **GPU纹理上传** |||
| TIME_BUDGET | 4ms/帧 | 每帧最大上传时间 |
| TILE_SIZE | 254px | 瓦片大小 |
| **GalleryBitmapPool** |||
| CAPACITY_BYTES | 20MB | 总容量 |
| POOL_SIZE | 3 | 子池数量 |
| **LruCache** |||
| DATA_CACHE_SIZE | 1000 | 媒体项缓存数 |
| **ImageCacheService** |||
| MAX_ENTRIES | 5000 | 磁盘缓存最大条目 |
| MAX_BYTES | 200MB | 磁盘缓存最大容量 |
| VERSION | 7 | 缓存版本号 |
| **ThreadPool** |||
| CORE_POOL_SIZE | 4 | 核心线程数 |
| MAX_POOL_SIZE | 8 | 最大线程数 |
| **AlbumSlidingWindow** |||
| CACHE_SIZE | 96 | 槽位缓存数 |
| JOB_LIMIT | 2 | 缩略图并发加载数 |
| **DecodeUtils** |||
| MAX_PIXEL_COUNT | 640000 | 最大解码像素数 |

---

## 十一、核心文件清单

| 文件 | 职责 |
|------|------|
| [GalleryAppImpl.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/GalleryAppImpl.java) | 应用初始化、组件创建 |
| [ImageCacheService.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/data/ImageCacheService.java) | 磁盘缓存服务 |
| [BlobCache.java](file:///d:/code/gallery2/Gallery2/gallerycommon/src/com/android/gallery3d/common/BlobCache.java) | 磁盘缓存实现 |
| [GalleryBitmapPool.java](file:///d:/code/gallery2/Gallery2/src/com/android/photos/data/GalleryBitmapPool.java) | 位图复用池 |
| [SparseArrayBitmapPool.java](file:///d:/code/gallery2/Gallery2/src/com/android/photos/data/SparseArrayBitmapPool.java) | 稀疏数组位图池 |
| [LruCache.java](file:///d:/code/gallery2/Gallery2/gallerycommon/src/com/android/gallery3d/common/LruCache.java) | LRU内存缓存 |
| [AlbumSlidingWindow.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/AlbumSlidingWindow.java) | 滑动窗口管理 |
| [AlbumDataLoader.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumDataLoader.java) | 数据加载器 |
| [DecodeUtils.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/data/DecodeUtils.java) | 解码工具 |
| [BitmapLoader.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/BitmapLoader.java) | 位图加载基类 |
| [TiledTexture.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/TiledTexture.java) | 瓦片纹理 |
| [BitmapTexture.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/glrenderer/BitmapTexture.java) | 位图纹理 |
| [AlbumSlotRenderer.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/AlbumSlotRenderer.java) | 宫格渲染器 |
| [SlotView.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/ui/SlotView.java) | 宫格视图 |
| [AlbumPage.java](file:///d:/code/gallery2/Gallery2/src/com/android/gallery3d/app/AlbumPage.java) | 相册页面 |

---

本文档详细分析了Gallery2的冷启动缓存分层处理机制，包括四级缓存架构、初始化流程、数据加载链路和渲染流程。
