# Android 完整学习笔记 - Part 5

> 包含详细答案 + 代码示例，适合深度学习和理解
>
> 共 32 题：Android UI补充（9题）+ 第三方框架（3题）+ 综合技术（6题）+ 系统SDK（5题）+ Kotlin补充（4题）+ 其他（5题）

---

## 📌 Android UI 绘制补充

### 1. RecyclerView 的缓存机制（四级缓存）

#### 详细答案

RecyclerView 有四级缓存，从快到慢依次：

| 缓存层级 | 名称 | 容量 | 说明 |
|---------|------|------|------|
| 第一级 | mAttachedScrap | 无限制 | 屏幕内正在显示的 ViewHolder |
| 第二级 | mCachedViews | 默认 2 | 滑出屏幕但未被回收的 ViewHolder（保留数据） |
| 第三级 | ViewCacheExtension | 自定义 | 开发者自定义缓存（较少使用） |
| 第四级 | RecycledViewPool | 每类型5个 | 最终回收池，复用前需重新 onBindViewHolder |

**滑动回收流程：**
```
Item 滑出屏幕
    ↓
放入 mCachedViews（保留绑定数据，可直接复用）
    ↓ mCachedViews 满了
放入 RecycledViewPool（清除绑定数据）

新 Item 滑入
    ↓ 先查 mAttachedScrap
    ↓ 再查 mCachedViews（命中则不需要 onBind）
    ↓ 再查 ViewCacheExtension
    ↓ 再查 RecycledViewPool（需要 onBind）
    ↓ 都没有则 onCreateViewHolder + onBindViewHolder
```

#### 代码示例

```java
public class RecyclerViewCacheDemo {

    public void configureCache(RecyclerView recyclerView) {
        // 1. 设置 mCachedViews 容量（默认2，适当增大减少 onBind 调用）
        recyclerView.setItemViewCacheSize(4);

        // 2. 配置 RecycledViewPool
        RecyclerView.RecycledViewPool pool = new RecyclerView.RecycledViewPool();
        pool.setMaxRecycledViews(0, 20);  // type=0 的 ViewHolder 最多缓存 20 个
        recyclerView.setRecycledViewPool(pool);

        // 3. 多个 RecyclerView 共享 RecycledViewPool（列表内嵌列表场景）
        RecyclerView inner1 = null, inner2 = null;
        RecyclerView.RecycledViewPool sharedPool = new RecyclerView.RecycledViewPool();
        inner1.setRecycledViewPool(sharedPool);
        inner2.setRecycledViewPool(sharedPool);
    }

    // 4. 自定义 ViewCacheExtension（第三级缓存）
    public RecyclerView.ViewCacheExtension createBannerCache() {
        return new RecyclerView.ViewCacheExtension() {
            private final Map<Integer, RecyclerView.ViewHolder> bannerCache = new HashMap<>();

            @Override
            public View getViewForPositionAndType(
                    RecyclerView.Recycler recycler, int position, int type) {
                if (type == ITEM_TYPE_BANNER) {
                    RecyclerView.ViewHolder holder = bannerCache.get(position);
                    return holder != null ? holder.itemView : null;
                }
                return null;
            }
        };
    }
}
```

#### 核心要点

```
四级缓存命中顺序：
mAttachedScrap → mCachedViews → ViewCacheExtension → RecycledViewPool

mCachedViews 命中：不需要 onBind（数据还在）
RecycledViewPool 命中：需要 onBind（数据已清除）

优化建议：
├─ setItemViewCacheSize(4)：增大二级缓存
├─ 嵌套 RecyclerView 共享 RecycledViewPool
├─ setRecycledViewPool().setMaxRecycledViews()：增大池容量
└─ setHasStableIds(true)：ItemId 稳定时进一步优化
```

---

### 2. Fragment 生命周期

#### 详细答案

Fragment 生命周期与 Activity 深度绑定，比 Activity 多了 `onAttach`、`onCreateView`、`onDestroyView`、`onDetach` 四个回调。

**完整生命周期：**
```
onAttach(Activity)        → Fragment 与 Activity 建立关联
onCreate()                → 初始化 Fragment（非 UI）
onCreateView()            → 创建并返回 View
onViewCreated()           → View 创建完毕（推荐在此初始化 View）
onActivityCreated()       → Activity.onCreate() 完成（已废弃，用 onViewCreated）
onStart()                 → Fragment 可见
onResume()                → Fragment 可交互
    ---- 用户操作 ----
onPause()                 → Fragment 失去焦点
onStop()                  → Fragment 不可见
onDestroyView()           → View 被销毁（Fragment 实例保留）
onDestroy()               → Fragment 销毁
onDetach()                → 与 Activity 解除关联
```

**Fragment 和 Activity 的生命周期对比：**
```
启动：
Activity.onCreate → Fragment.onAttach → Fragment.onCreate →
Fragment.onCreateView → Fragment.onViewCreated →
Activity.onStart → Fragment.onStart →
Activity.onResume → Fragment.onResume

返回退出：
Fragment.onPause → Activity.onPause →
Fragment.onStop → Activity.onStop →
Fragment.onDestroyView → Fragment.onDestroy → Fragment.onDetach →
Activity.onDestroy
```

#### 代码示例

```kotlin
class UserFragment : Fragment(R.layout.fragment_user) {

    // onAttach：可以拿到 Activity，建立通信接口
    override fun onAttach(context: Context) {
        super.onAttach(context)
        // 获取宿主 Activity 实现的接口
        if (context is OnUserActionListener) {
            listener = context
        }
    }

    // onCreate：做非 UI 初始化（ViewModel、参数接收）
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val userId = arguments?.getString(ARG_USER_ID) ?: ""
    }

    // onCreateView：创建布局（使用构造函数传入 layoutId 时可不重写）
    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentUserBinding.inflate(inflater, container, false)
        return binding.root
    }

    // onViewCreated：View 初始化（推荐在这里做 UI 操作）
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // ✅ 在这里初始化 View、设置监听器、观察 LiveData
        setupViews()
        observeViewModel()
    }

    // onDestroyView：清理 View 引用，防止内存泄漏
    private var _binding: FragmentUserBinding? = null
    private val binding get() = _binding!!

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null  // ⚠️ 必须置空，否则内存泄漏（Fragment 实例比 View 活得久）
    }

    override fun onDetach() {
        super.onDetach()
        listener = null  // 清理接口引用
    }

    companion object {
        private const val ARG_USER_ID = "userId"

        fun newInstance(userId: String) = UserFragment().apply {
            arguments = bundleOf(ARG_USER_ID to userId)
        }
    }
}
```

#### 核心要点

```
关键注意事项：
├─ onCreateView vs onViewCreated：
│   onCreateView 返回 View，onViewCreated 做初始化
│   推荐在 onViewCreated 中初始化 UI
│
├─ _binding = null（onDestroyView 中）：
│   Fragment 实例生命周期 > View 生命周期
│   不置空会导致内存泄漏
│
├─ getViewLifecycleOwner()：
│   观察 LiveData 时用 viewLifecycleOwner（不是 this）
│   避免 View 销毁后仍收到回调
│
└─ add vs replace：
    add：Fragment 不销毁，仅 onPause/onStop
    replace：Fragment 走完整销毁流程（回退栈除外）
```

---

### 3. SurfaceView 和 TextureView 的区别

#### 详细答案

| 特性 | SurfaceView | TextureView |
|------|------------|-------------|
| **渲染线程** | 独立线程（不阻塞 UI） | 主线程（硬件加速） |
| **层级** | 独立 Surface，不在 View 树中 | 在 View 树中 |
| **动画/变换** | ❌ 不支持（位置固定） | ✅ 支持旋转/缩放/透明度 |
| **内存** | 低 | 高（持有 SurfaceTexture） |
| **API** | API 1+ | API 14+ |
| **适用场景** | 高帧率视频、游戏 | 需要变换的视频、相机预览 |

#### 代码示例

```java
// SurfaceView 使用（游戏/视频播放）
public class GameSurfaceView extends SurfaceView implements SurfaceHolder.Callback {
    private SurfaceHolder holder;
    private GameThread gameThread;

    public GameSurfaceView(Context context) {
        super(context);
        holder = getHolder();
        holder.addCallback(this);
        setZOrderOnTop(false);  // false: 在其他 View 后面; true: 置顶
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        gameThread = new GameThread(holder);
        gameThread.setRunning(true);
        gameThread.start();
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {}

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        gameThread.setRunning(false);
        try { gameThread.join(); } catch (InterruptedException e) { e.printStackTrace(); }
    }

    // 在独立线程中绘制（不阻塞主线程）
    class GameThread extends Thread {
        private final SurfaceHolder holder;
        private boolean running = false;

        public GameThread(SurfaceHolder holder) { this.holder = holder; }

        public void setRunning(boolean running) { this.running = running; }

        @Override
        public void run() {
            while (running) {
                Canvas canvas = null;
                try {
                    canvas = holder.lockCanvas();  // 锁定画布
                    synchronized (holder) {
                        draw(canvas);  // 在独立线程绘制
                    }
                } finally {
                    if (canvas != null) holder.unlockCanvasAndPost(canvas);  // 提交
                }
            }
        }

        private void draw(Canvas canvas) {
            canvas.drawColor(Color.BLACK);
            // 绘制游戏元素...
        }
    }
}

// TextureView 使用（需要动画变换的相机预览）
public class CameraPreviewActivity extends AppCompatActivity
        implements TextureView.SurfaceTextureListener {

    private TextureView textureView;
    private Camera camera;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        textureView = new TextureView(this);
        textureView.setSurfaceTextureListener(this);
        setContentView(textureView);
    }

    @Override
    public void onSurfaceTextureAvailable(SurfaceTexture surface, int width, int height) {
        camera = Camera.open();
        try {
            camera.setPreviewTexture(surface);
            camera.startPreview();
        } catch (IOException e) { e.printStackTrace(); }

        // TextureView 支持动画（SurfaceView 不支持）
        textureView.setRotation(45);   // 旋转
        textureView.setScaleX(0.5f);   // 缩放
        textureView.setAlpha(0.8f);    // 透明度
    }

    @Override
    public void onSurfaceTextureSizeChanged(SurfaceTexture surface, int w, int h) {}

    @Override
    public boolean onSurfaceTextureDestroyed(SurfaceTexture surface) {
        camera.stopPreview();
        camera.release();
        return true;
    }

    @Override
    public void onSurfaceTextureUpdated(SurfaceTexture surface) {}
}
```

#### 核心要点

```
选择建议：
├─ 高帧率渲染（游戏/视频播放）→ SurfaceView
│   独立 Surface，独立线程绘制，性能最好
│
└─ 需要 View 变换（动画/旋转/透明度）→ TextureView
   在 View 树中，支持所有 View 动画

SurfaceView 的"挖洞"原理：
└─ SurfaceView 在 Window 上打了一个"洞"，让底层 Surface 透出来显示
   所以它不参与 View 的绘制流程，也不支持变换
```

---

### 4. LayoutInflater.inflate 的工作原理

#### 详细答案

`inflate()` 将 XML 布局文件解析成 View 对象，核心是 **XmlPullParser + 反射**。

**流程：**
```
inflate(layoutId, parent, attachToRoot)
    ↓
XmlPullParser 解析 XML 文件
    ↓
读取根标签名（如 "LinearLayout"）
    ↓
通过 createView() 用反射创建 View 实例
    ↓
递归处理子标签
    ↓
返回根 View
```

#### 代码示例

```java
// inflate 的三种调用方式（重要！）
public class InflateDemo {

    public void inflateUsage(Context context, ViewGroup parent) {
        LayoutInflater inflater = LayoutInflater.from(context);

        // 方式1：inflate(id, null)
        // → 根 View 的 layout_width/height 属性会失效（没有父容器提供参数）
        View view1 = inflater.inflate(R.layout.item, null);

        // 方式2：inflate(id, parent, false) ✅ 推荐
        // → 根 View 的 layout 属性有效（parent 提供测量依据），但不添加到 parent
        View view2 = inflater.inflate(R.layout.item, parent, false);

        // 方式3：inflate(id, parent, true)
        // → 直接将 View 添加到 parent，返回 parent（不是 item）
        View view3 = inflater.inflate(R.layout.item, parent, true);

        // RecyclerView Adapter 中正确写法
        // ✅ 使用 (parent, false)
        View itemView = inflater.inflate(R.layout.item_user, parent, false);
    }

    // 特殊标签处理
    public void specialTags(Context context, ViewGroup parent) {
        LayoutInflater inflater = LayoutInflater.from(context);

        // <merge>：减少布局层级（必须传入 parent，且 attachToRoot=true）
        inflater.inflate(R.layout.merge_layout, parent, true);

        // <include>：复用布局（inflate 时正常处理）

        // <ViewStub>：延迟加载（stub.inflate() 时才真正创建 View）
        ViewStub stub = new ViewStub(context);
        stub.setLayoutResource(R.layout.heavy_layout);
        View inflated = stub.inflate();  // 首次调用才 inflate
    }
}
```

```java
// LayoutInflater 自定义 Factory（Hook View 创建过程）
public class ThemeApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        // 通过 Factory2 可以拦截所有 View 的创建
        // 换肤框架（如 AndroidX AppCompat）就是这么实现的
    }
}

// AppCompatActivity 内部实现
// activity.getLayoutInflater().setFactory2(new AppCompatDelegateImpl());
// 当解析到 <Button> 时，创建 AppCompatButton 而非原生 Button
```

#### 核心要点

```
inflate 三参数的区别：
├─ inflate(id, null)：layout_width/height 失效（无父容器参考）
├─ inflate(id, parent, false)：layout 属性有效，不添加到 parent（RecyclerView Adapter推荐）
└─ inflate(id, parent, true)：添加到 parent，返回 parent

XmlPullParser 解析流程：
1. 读取 XML 文件（从 res 目录）
2. 识别标签名（如 "LinearLayout"）
3. 反射调用 Constructor(Context, AttributeSet) 创建对象
4. 递归处理子 View
5. 应用 LayoutParams

性能优化：
└─ 使用 ViewBinding/DataBinding 替代 inflate（编译时生成，无反射）
```

---

### 5. View.post() 为何能获取宽高

#### 详细答案

在 `onCreate()` 中直接调用 `view.getWidth()` 返回 0，因为 View 还未完成测量。
`view.post(Runnable)` 能获取宽高，**原理是将 Runnable 延迟到 View 绘制完成后执行**。

**源码分析：**
```
View.post(Runnable)
    ↓
如果 mAttachInfo != null（已 attachToWindow）
    → 直接 post 到主线程 Handler
如果 mAttachInfo == null（还未 attachToWindow）
    → 存入 HandlerActionQueue（等待队列）

View.dispatchAttachedToWindow()（View 被添加到 Window 时调用）
    ↓
执行 HandlerActionQueue 中缓存的所有 Runnable
    ↓
此时 View 已完成测量布局，可以获取宽高
```

#### 代码示例

```kotlin
class WidthHeightDemo : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val view = binding.myView

        // ❌ 直接获取：返回 0（View 未测量）
        Log.d("View", "直接获取 width=${view.width}")  // 0

        // ✅ 方式1：View.post()
        view.post {
            Log.d("View", "post width=${view.width}")  // 正确值
        }

        // ✅ 方式2：ViewTreeObserver.OnGlobalLayoutListener
        view.viewTreeObserver.addOnGlobalLayoutListener(object :
            ViewTreeObserver.OnGlobalLayoutListener {
            override fun onGlobalLayout() {
                // 每次布局完成都会调用，需要手动移除
                view.viewTreeObserver.removeOnGlobalLayoutListener(this)
                Log.d("View", "GlobalLayout width=${view.width}")
            }
        })

        // ✅ 方式3：View.measure() 手动测量（不依赖 View 树）
        val widthSpec = View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED)
        val heightSpec = View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED)
        view.measure(widthSpec, heightSpec)
        val measuredWidth = view.measuredWidth

        // ✅ 方式4：doOnLayout 扩展函数（KTX）
        view.doOnLayout {
            Log.d("View", "doOnLayout width=${it.width}")
        }

        // ✅ 方式5：doOnPreDraw（绘制前）
        view.doOnPreDraw {
            Log.d("View", "doOnPreDraw width=${it.width}")
        }
    }
}
```

#### 核心要点

```
为什么 onCreate 中宽高为 0？
└─ setContentView 只是将 View 添加到 DecorView
   measure/layout 在 handleResumeActivity 之后的 traversal 中执行
   所以 onCreate 时 View 还没经过 measure

View.post() 原理：
├─ attach 之前：缓存到 HandlerActionQueue
├─ attach 之后：dispatchAttachedToWindow 时执行缓存队列
└─ 此时已完成 measure/layout，可以获取宽高

获取宽高的 4 种方式：
├─ view.post { }：最简单
├─ OnGlobalLayoutListener：可感知每次布局变化
├─ view.measure()：手动强制测量
└─ doOnLayout { }：KTX 扩展，本质同 OnGlobalLayoutListener
```

---

### 6. 同步屏障（MessageQueue SyncBarrier）

#### 详细答案

同步屏障是 `MessageQueue` 中一种特殊消息，**拦截所有同步消息，让异步消息优先执行**。

**使用场景：** ViewRootImpl 在请求 UI 渲染时，插入同步屏障，让 Choreographer 的异步帧绘制消息优先处理，保证 UI 流畅。

**三种 Message 类型：**
- **同步消息**（普通 Message）：按时间顺序执行
- **异步消息**（setAsynchronous(true)）：遇到屏障时可跳过屏障优先执行
- **屏障消息**（target == null 的 Message）：插入屏障

#### 代码示例

```java
// 同步屏障的工作原理（源码分析）
public class SyncBarrierDemo {

    public void explainSyncBarrier() {
        /*
        MessageQueue.next() 取消息的逻辑：

        for (;;) {
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 遇到屏障（target == null）
                // 跳过所有同步消息，找第一个异步消息
                do {
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            // 执行异步消息 msg
        }
        */

        // ViewRootImpl 中如何使用同步屏障：
        /*
        void scheduleTraversals() {
            if (!mTraversalScheduled) {
                mTraversalScheduled = true;
                // 1. 插入同步屏障（阻塞后续同步消息）
                mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();

                // 2. 发送异步消息（帧回调）
                Choreographer.getInstance().postCallback(
                    Choreographer.CALLBACK_TRAVERSAL,
                    mTraversalRunnable,   // 实际执行 measure/layout/draw
                    null
                );
            }
        }

        void doTraversal() {
            if (mTraversalScheduled) {
                mTraversalScheduled = false;
                // 3. 移除同步屏障
                mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

                performTraversals();  // 执行绘制
            }
        }
        */
    }

    // 发送异步消息（需要 API 22+）
    public void sendAsyncMessage(Handler handler) {
        Message msg = Message.obtain();
        msg.setAsynchronous(true);  // 标记为异步消息
        handler.sendMessage(msg);
    }
}
```

#### 核心要点

```
同步屏障机制：
├─ 屏障消息：target == null 的特殊 Message
├─ 作用：阻塞同步消息，让异步消息先执行
└─ 移除：removeSyncBarrier() 必须成对调用，否则主线程无法处理同步消息

ViewRootImpl 使用流程：
1. scheduleTraversals() → postSyncBarrier() 插入屏障
2. Choreographer 收到 VSync → 发送异步帧绘制消息
3. 异步消息跳过屏障优先执行
4. doTraversal() → removeSyncBarrier() 移除屏障
5. 后续同步消息恢复处理

⚠️ 注意：postSyncBarrier() 是隐藏 API，普通开发者不直接使用
```

---

### 7. Fragment 懒加载

#### 详细答案

Fragment 懒加载是指 **Fragment 真正对用户可见时才加载数据**，避免 ViewPager 预加载导致不可见的 Fragment 也请求网络。

**实现方案演进：**
- **旧方案**（API < 28）：重写 `setUserVisibleHint()`
- **新方案**（API 28+）：`setMaxLifecycle()` 控制生命周期
- **推荐方案**：ViewPager2 + FragmentStateAdapter，天然懒加载

#### 代码示例

```kotlin
// 方案1：旧方案 setUserVisibleHint（已废弃，ViewPager1 中使用）
abstract class LazyFragment : Fragment() {
    private var isLoaded = false

    @Deprecated("Use setMaxLifecycle instead")
    override fun setUserVisibleHint(isVisibleToUser: Boolean) {
        super.setUserVisibleHint(isVisibleToUser)
        if (isVisibleToUser && !isLoaded && isResumed) {
            lazyLoad()
            isLoaded = true
        }
    }

    override fun onResume() {
        super.onResume()
        if (userVisibleHint && !isLoaded) {
            lazyLoad()
            isLoaded = true
        }
    }

    abstract fun lazyLoad()
}

// 方案2：新方案 FragmentPagerAdapter + setMaxLifecycle（ViewPager1 中推荐）
class LazyFragmentPagerAdapter(
    fm: FragmentManager,
    private val fragments: List<Fragment>
) : FragmentPagerAdapter(fm, BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
    // BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT：只有当前 Fragment 才会走到 onResume
    // 其他 Fragment 最多走到 onStart（Lifecycle.State.STARTED）

    override fun getItem(position: Int) = fragments[position]
    override fun getCount() = fragments.size
}

// Fragment 中：onResume 时才加载数据（其他 Fragment 不会走到 onResume）
class UserFragment : Fragment() {
    private var isLoaded = false

    override fun onResume() {
        super.onResume()
        if (!isLoaded) {
            loadData()
            isLoaded = true
        }
    }
}

// 方案3：ViewPager2 + FragmentStateAdapter（最推荐）
class MyFragmentStateAdapter(
    fragmentActivity: FragmentActivity,
    private val fragments: List<Fragment>
) : FragmentStateAdapter(fragmentActivity) {
    override fun getItemCount() = fragments.size
    override fun createFragment(position: Int) = fragments[position]
    // ViewPager2 默认每次只有当前 Fragment 处于 RESUMED 状态
    // 天然支持懒加载，无需额外处理
}

// Fragment 中直接在 onResume 加载
class ContentFragment : Fragment() {
    override fun onResume() {
        super.onResume()
        // ViewPager2 中，只有当前页面才会走到 onResume
        viewModel.loadData()
    }
}
```

#### 核心要点

```
懒加载演进：
├─ ViewPager1 旧方案：setUserVisibleHint()（已废弃）
├─ ViewPager1 新方案：BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT
└─ ViewPager2：天然懒加载，在 onResume 加载即可

ViewPager vs ViewPager2：
├─ ViewPager：预加载相邻 Fragment，默认缓存1页（offscreenPageLimit）
└─ ViewPager2：基于 RecyclerView，默认不预加载，更节省内存
```

---

### 8. ViewDragHelper 工作原理

#### 详细答案

`ViewDragHelper` 是 Android 提供的辅助类，简化拖拽子 View 的实现，常用于侧滑菜单、抽屉效果。

#### 代码示例

```java
public class DragLayout extends FrameLayout {
    private ViewDragHelper dragHelper;

    public DragLayout(Context context) {
        super(context);
        dragHelper = ViewDragHelper.create(this, 1.0f, new ViewDragHelper.Callback() {

            // 判断哪些 View 可以拖拽（返回 true 则该 View 可以拖拽）
            @Override
            public boolean tryCaptureView(View child, int pointerId) {
                return child.getId() == R.id.drag_view;
            }

            // 水平方向拖拽范围（返回 0 则禁止水平拖拽）
            @Override
            public int getViewHorizontalDragRange(View child) {
                return getMeasuredWidth() - child.getMeasuredWidth();
            }

            // 垂直方向拖拽范围
            @Override
            public int getViewVerticalDragRange(View child) {
                return getMeasuredHeight() - child.getMeasuredHeight();
            }

            // 限制水平移动范围（返回值为最终的 left 坐标）
            @Override
            public int clampViewPositionHorizontal(View child, int left, int dx) {
                // 限制在父容器范围内
                int leftBound = getPaddingLeft();
                int rightBound = getWidth() - child.getWidth();
                return Math.max(leftBound, Math.min(left, rightBound));
            }

            // 限制垂直移动范围
            @Override
            public int clampViewPositionVertical(View child, int top, int dy) {
                int topBound = getPaddingTop();
                int bottomBound = getHeight() - child.getHeight();
                return Math.max(topBound, Math.min(top, bottomBound));
            }

            // 拖拽结束后处理（松手时自动吸附）
            @Override
            public void onViewReleased(View releasedChild, float xvel, float yvel) {
                // 松手后自动归位到左侧
                dragHelper.settleCapturedViewAt(0, releasedChild.getTop());
                invalidate();
            }
        });
    }

    // 必须在 onInterceptTouchEvent 和 onTouchEvent 中转发事件
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return dragHelper.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        dragHelper.processTouchEvent(event);
        return true;
    }

    // settleCapturedViewAt 内部使用 Scroller，需要通过 computeScroll 驱动
    @Override
    public void computeScroll() {
        if (dragHelper.continueSettling(true)) {
            invalidate();  // 继续动画
        }
    }
}
```

#### 核心要点

```
ViewDragHelper 关键回调：
├─ tryCaptureView()：决定哪个 View 可被拖拽
├─ clampViewPositionHorizontal/Vertical()：限制移动范围
├─ getViewHorizontalDragRange()：声明水平拖拽范围（>0 才能水平拖）
└─ onViewReleased()：松手后处理（归位动画）

必须转发事件：
└─ onInterceptTouchEvent → dragHelper.shouldInterceptTouchEvent()
   onTouchEvent → dragHelper.processTouchEvent()

settleCapturedViewAt 驱动：
└─ 内部用 Scroller，需要 computeScroll() + invalidate() 驱动动画
```

---

### 9. RecyclerView.Adapter 数据刷新方式

#### 详细答案

| 方法 | 作用 | 性能 |
|------|------|------|
| `notifyDataSetChanged()` | 全量刷新 | 最差（全部重绘） |
| `notifyItemChanged(pos)` | 单条刷新 | 好 |
| `notifyItemInserted(pos)` | 单条插入 | 好 |
| `notifyItemRemoved(pos)` | 单条删除 | 好 |
| `notifyItemMoved(from, to)` | 移动 | 好 |
| `notifyItemRangeChanged()` | 范围刷新 | 中 |
| **DiffUtil** | 自动计算差异 | 最佳 |
| **AsyncListDiffer** | 异步差异计算 | 最佳（后台线程） |

#### 代码示例

```kotlin
// 1. DiffUtil（推荐）
class UserDiffCallback(
    private val oldList: List<User>,
    private val newList: List<User>
) : DiffUtil.Callback() {

    override fun getOldListSize() = oldList.size
    override fun getNewListSize() = newList.size

    // 是否是同一个 Item（通常比较 ID）
    override fun areItemsTheSame(oldPos: Int, newPos: Int) =
        oldList[oldPos].id == newList[newPos].id

    // 内容是否相同（areItemsTheSame 返回 true 时才调用）
    override fun areContentsTheSame(oldPos: Int, newPos: Int) =
        oldList[oldPos] == newList[newPos]

    // 返回变更的 payload（部分刷新，避免整条闪烁）
    override fun getChangePayload(oldPos: Int, newPos: Int): Any? {
        val oldUser = oldList[oldPos]
        val newUser = newList[newPos]
        return if (oldUser.name != newUser.name) "name_changed" else null
    }
}

class UserAdapter : RecyclerView.Adapter<UserViewHolder>() {
    private var userList = emptyList<User>()

    fun submitList(newList: List<User>) {
        val diffResult = DiffUtil.calculateDiff(UserDiffCallback(userList, newList))
        userList = newList
        diffResult.dispatchUpdatesTo(this)  // 自动调用对应的 notify 方法
    }

    // 处理 payload 局部刷新（只更新名字，不整条刷新）
    override fun onBindViewHolder(holder: UserViewHolder, position: Int, payloads: List<Any>) {
        if (payloads.isEmpty()) {
            onBindViewHolder(holder, position)  // 全量刷新
        } else {
            if (payloads.contains("name_changed")) {
                holder.binding.tvName.text = userList[position].name  // 只更新名字
            }
        }
    }
}

// 2. ListAdapter（内置 DiffUtil，更简洁）
class BetterUserAdapter : ListAdapter<User, UserViewHolder>(
    object : DiffUtil.ItemCallback<User>() {
        override fun areItemsTheSame(old: User, new: User) = old.id == new.id
        override fun areContentsTheSame(old: User, new: User) = old == new
    }
) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder { TODO() }
    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

// 使用：submitList 会在后台线程计算 Diff，主线程更新 UI
adapter.submitList(newUserList)
```

---

## 📌 第三方框架

### 10. EventBus 的实现原理

#### 详细答案

EventBus 基于**发布-订阅模式**，通过**反射**查找订阅方法，通过**注解处理器（APT）** 在编译期生成索引加速查找。

**核心流程：**
```
register(subscriber)
    ↓
反射遍历 subscriber 所有方法
找到 @Subscribe 注解的方法
按事件类型存入 Map<Class, List<Subscription>>

post(event)
    ↓
根据 event.getClass() 从 Map 取订阅列表
    ↓
根据 threadMode 切换线程
    ↓
反射调用订阅方法
```

#### 代码示例

```java
// 1. 基本使用
public class LoginActivity extends AppCompatActivity {

    @Override
    protected void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);  // 注册
    }

    @Override
    protected void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);  // 注销（必须！否则内存泄漏）
    }

    // 订阅方法（必须是 public，参数只能有一个）
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onLoginEvent(LoginEvent event) {
        // 在主线程执行
        updateUI(event.getUser());
    }

    @Subscribe(threadMode = ThreadMode.BACKGROUND, sticky = true)
    public void onDataEvent(DataEvent event) {
        // 在后台线程执行，可以接收粘性事件
        processData(event.getData());
    }
}

// 2. 发布事件
public class LoginPresenter {
    public void onLoginSuccess(User user) {
        EventBus.getDefault().post(new LoginEvent(user));  // 普通事件

        // 粘性事件（postSticky）：先发布后注册的订阅者也能收到
        EventBus.getDefault().postSticky(new UserInfoEvent(user));
    }
}

// 3. EventBus ThreadMode
/*
POSTING：在发送事件的线程执行（默认）
MAIN：在主线程执行（发送者是主线程则直接调用，否则切换）
MAIN_ORDERED：在主线程排队执行（总是通过 Handler post）
BACKGROUND：在后台线程执行（发送者是后台线程则直接调用，否则用线程池）
ASYNC：总是在新的异步线程执行（用线程池）
*/
```

#### 核心要点

```
EventBus 工作原理：
├─ 注册时：反射扫描 @Subscribe 方法，存入 subscriptionsByEventType Map
├─ 发布时：根据 event 类型找到订阅列表，按 threadMode 执行
└─ 注销时：从 Map 中移除该订阅者的所有方法

APT（注解处理器）优化：
└─ 编译期生成 MyEventBusIndex，避免运行时反射扫描（提升启动速度）
   EventBus.builder().addIndex(new MyEventBusIndex()).build()

粘性事件原理：
└─ postSticky 时存入 stickyEvents Map
   新订阅者 register 时，检查是否有匹配的粘性事件，有则立即投递

注意事项：
├─ 必须在 onStop/onDestroy 中 unregister（否则内存泄漏）
├─ 订阅方法必须 public（反射调用）
└─ 不推荐在大型项目中大量使用（事件流向难以追踪）
```

---

### 11. LeakCanary 的工作原理

#### 详细答案

LeakCanary 基于 **WeakReference + ReferenceQueue + 堆转储分析** 检测内存泄漏。

**检测流程：**
```
Activity/Fragment onDestroy
    ↓
LeakCanary 创建 WeakReference<Activity> + ReferenceQueue
    ↓
等待 5 秒（等 GC）
    ↓
检查 WeakReference 是否入队 ReferenceQueue
    ↓
没有入队（说明 Activity 对象仍存活）
    ↓
手动触发 GC，再等待
    ↓
仍未入队 → 判定为内存泄漏
    ↓
dump heap（获取内存快照 .hprof）
    ↓
Shark 库分析 .hprof 找到引用链
    ↓
通知用户（Notification）
```

#### 代码示例

```groovy
// 1. 添加依赖（只在 debug 包中）
dependencies {
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.12'
}
// LeakCanary 2.x 无需手动初始化（通过 ContentProvider 自动初始化）
```

```kotlin
// 2. LeakCanary 核心原理实现（简化版）
class LeakWatcher {
    private val watchedObjects = mutableMapOf<String, WeakReference<Any>>()
    private val queue = ReferenceQueue<Any>()

    fun watch(watchedObject: Any, description: String) {
        val key = UUID.randomUUID().toString()
        // 创建弱引用，关联 ReferenceQueue
        watchedObjects[key] = WeakReference(watchedObject, queue)

        // 5秒后检查是否泄漏
        checkRetainedAfterDelay(key, description)
    }

    private fun checkRetainedAfterDelay(key: String, description: String) {
        Handler(Looper.getMainLooper()).postDelayed({
            // 清理已被回收的引用
            var ref = queue.poll()
            while (ref != null) {
                watchedObjects.values.remove(ref)
                ref = queue.poll()
            }

            // 如果还在 map 中，说明未被 GC 回收
            if (watchedObjects.containsKey(key)) {
                // 触发 GC
                Runtime.getRuntime().gc()
                // 再次检查
                Handler(Looper.getMainLooper()).postDelayed({
                    if (watchedObjects.containsKey(key)) {
                        // 确认内存泄漏，dump heap
                        dumpHeap(description)
                    }
                }, 2000)
            }
        }, 5000)
    }
}

// 3. LeakCanary 如何监听 Activity：
// Application.registerActivityLifecycleCallbacks()
// 监听 onActivityDestroyed → watch(activity)
```

#### 核心要点

```
LeakCanary 工作原理：
1. 监听 Activity/Fragment 销毁（ActivityLifecycleCallbacks）
2. 销毁后用 WeakReference + ReferenceQueue 跟踪对象
3. 等 GC 后对象仍存活 → 泄漏
4. Heap Dump → Shark 分析引用链 → 展示泄漏路径

WeakReference + ReferenceQueue 原理：
├─ WeakReference：GC 时无论内存是否不足，都会回收弱引用对象
└─ ReferenceQueue：对象被 GC 时，其 WeakReference 会自动入队
   检查 ReferenceQueue 中是否有该对象的引用 → 判断是否已被回收

Shark 库：
└─ Kotlin 实现的 .hprof 文件分析库，比旧版 haha 更快更省内存
```

---

### 12. RxJava 背压（Backpressure）原理

#### 详细答案

背压是指**生产者发射速度 > 消费者处理速度**时的流量控制策略。

**RxJava 2 解决方案：**
- `Observable`：不支持背压（缓冲区无限增长，可能 OOM）
- `Flowable`：支持背压，有缓冲区大小限制（默认 128）

#### 代码示例

```java
// 背压问题演示
public class BackpressureDemo {

    // ❌ Observable 不处理背压（可能 OOM）
    public void withoutBackpressure() {
        Observable.interval(1, TimeUnit.MILLISECONDS)   // 快速生产
            .observeOn(Schedulers.computation())
            .subscribe(item -> {
                Thread.sleep(100);  // 慢速消费
                System.out.println(item);
            });
        // 结果：MissingBackpressureException 或 OOM
    }

    // ✅ Flowable + 背压策略
    public void withBackpressure() {
        Flowable.interval(1, TimeUnit.MILLISECONDS)
            .onBackpressureDrop()         // 策略1：丢弃来不及处理的数据
            // .onBackpressureBuffer(100) // 策略2：缓冲（超出缓冲区则报错）
            // .onBackpressureLatest()    // 策略3：只保留最新的数据
            .observeOn(Schedulers.computation())
            .subscribe(item -> {
                Thread.sleep(100);
                System.out.println(item);
            });
    }

    // 背压策略说明
    public void backpressureStrategies() {
        /*
        BackpressureStrategy.ERROR：
            下游处理不过来时抛异常（MissingBackpressureException）

        BackpressureStrategy.BUFFER：
            缓冲所有数据（内存可能溢出）

        BackpressureStrategy.DROP：
            丢弃来不及处理的数据（保留最早的）

        BackpressureStrategy.LATEST：
            只保留最新的数据（丢弃旧数据）

        BackpressureStrategy.MISSING：
            不处理（由下游自己处理，通常配合 onBackpressureXxx）
        */
        Flowable.create(emitter -> {
            for (int i = 0; i < 1000; i++) {
                emitter.onNext(i);
            }
            emitter.onComplete();
        }, BackpressureStrategy.DROP)
        .observeOn(Schedulers.io())
        .subscribe(System.out::println);
    }

    // Flowable 请求式背压（响应式拉取）
    public void requestBackpressure() {
        Flowable.range(1, 100)
            .subscribe(new Subscriber<Integer>() {
                Subscription subscription;

                @Override
                public void onSubscribe(Subscription s) {
                    subscription = s;
                    s.request(10);  // 先请求 10 个
                }

                @Override
                public void onNext(Integer integer) {
                    System.out.println(integer);
                    if (integer % 10 == 0) {
                        subscription.request(10);  // 处理完 10 个后再请求 10 个
                    }
                }

                @Override
                public void onError(Throwable t) {}

                @Override
                public void onComplete() {}
            });
    }
}
```

#### 核心要点

```
背压解决方案：
├─ Observable：无背压支持，生产速度不受控（适合 UI 事件等不会OOM的场景）
└─ Flowable：有背压支持，默认缓冲区 128（适合网络、数据库等数据流）

背压策略：
├─ DROP：丢弃多余数据（适合传感器数据，只要最新）
├─ BUFFER：缓冲（有 OOM 风险）
├─ LATEST：只保留最新（适合搜索框输入）
└─ ERROR：超出则报错

request(n) 响应式拉取：
└─ Subscriber 通过 subscription.request(n) 主动告诉上游需要多少数据
   实现真正的背压控制（上游按需生产）
```

---

## 📌 综合技术

### 13. 热修复原理

#### 详细答案

热修复让 App **不重新安装、不重启即可修复线上 Bug**，核心是修改 ClassLoader 的类加载顺序。

**主流方案：**

| 方案 | 原理 | 代表 |
|------|------|------|
| **类加载替换** | 修改 dexElements 数组顺序 | Tinker、Nuwa |
| **底层方法替换** | Native 替换 ArtMethod 指针 | AndFix、Sophix |
| **资源修复** | 重新创建 AssetManager | Tinker |

#### 代码示例

```java
// 类加载替换原理（Dex 插桩）
public class HotFixDemo {

    /**
     * Android ClassLoader 加载类的流程：
     *
     * PathClassLoader（App 使用）
     *     ↓
     * BaseDexClassLoader
     *     ↓ 包含
     * DexPathList
     *     ↓ 包含
     * dexElements[]（Element 数组）
     *
     * 类加载时：遍历 dexElements，找到第一个包含目标类的 dex 并加载
     *
     * 热修复思路：
     * 将修复了 Bug 的 dex 插入到 dexElements 数组最前面
     * → 优先加载修复版的类，原有 bug 类被跳过
     */
    public static void applyHotFix(Context context, String patchDexPath) {
        try {
            // 1. 获取当前 ClassLoader 的 dexElements
            ClassLoader classLoader = context.getClassLoader();
            Class<?> pathListClass = Class.forName("dalvik.system.DexPathList");

            Field pathListField = classLoader.getClass().getSuperclass()
                .getDeclaredField("pathList");
            pathListField.setAccessible(true);
            Object dexPathList = pathListField.get(classLoader);

            Field dexElementsField = pathListClass.getDeclaredField("dexElements");
            dexElementsField.setAccessible(true);
            Object[] dexElements = (Object[]) dexElementsField.get(dexPathList);

            // 2. 加载 patch dex 的 dexElements
            DexClassLoader patchLoader = new DexClassLoader(
                patchDexPath,
                context.getCacheDir().getAbsolutePath(),
                null,
                classLoader
            );
            Object patchPathList = pathListField.get(patchLoader);
            Object[] patchElements = (Object[]) dexElementsField.get(patchPathList);

            // 3. 将 patch dexElements 合并到原数组前面
            Object[] newElements = new Object[patchElements.length + dexElements.length];
            System.arraycopy(patchElements, 0, newElements, 0, patchElements.length);
            System.arraycopy(dexElements, 0, newElements, patchElements.length, dexElements.length);

            // 4. 替换 dexElements
            dexElementsField.set(dexPathList, newElements);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

#### 核心要点

```
热修复原理：
└─ Android ClassLoader 加载类时遍历 dexElements[]
   找到类就加载（不会重复加载）
   → 将 patch.dex 插入数组最前面，让修复版类优先被加载

两大流派：
1. Dex 替换（Tinker）：
   ├─ 稳定、兼容性好
   └─ 需要重启 App（类不能被替换，只能在下次启动生效）

2. 方法替换（AndFix/Sophix）：
   ├─ 即时生效（Native 层修改 ArtMethod 结构体）
   └─ 兼容性差（各厂商 ROM 定制 ArtMethod 结构不同）

CLASS_ISPREVERIFIED 问题（Dalvik 时代）：
└─ Dalvik 对类做了预校验，同一 dex 的类引用会被标记
   需要在每个类的构造函数中引用另一个 dex 的类来"打破"预校验
```

---

### 14. AOP 面向切面编程

#### 详细答案

AOP（Aspect-Oriented Programming）是一种编程范式，**在不修改原有代码的情况下，在方法执行的前后插入横切逻辑**。

**Android 中主流 AOP 方案：**

| 方案 | 实现方式 | 时机 |
|------|---------|------|
| **AspectJ** | 字节码插桩 | 编译期/加载期 |
| **动态代理** | Java Proxy | 运行时 |
| **Javassist/ASM** | 字节码操作 | 编译期（Gradle Plugin） |

#### 代码示例

```java
// AspectJ 使用示例（需要 gradle 插件支持）

// 1. 定义切面（Aspect）
@Aspect
public class LogAspect {

    // 切入点：匹配所有带 @LoginRequired 注解的方法
    @Pointcut("execution(@com.example.LoginRequired * *(..))")
    public void loginRequiredMethod() {}

    // 前置通知：方法执行前
    @Before("loginRequiredMethod()")
    public void checkLogin(JoinPoint joinPoint) throws Throwable {
        if (!UserManager.isLoggedIn()) {
            // 跳转到登录页
            startLoginActivity();
            throw new RuntimeException("需要登录");
        }
    }

    // 环绕通知：完全控制方法执行
    @Around("execution(* com.example..*Activity.*(..)) && @annotation(timeLog)")
    public Object logTime(ProceedingJoinPoint joinPoint, TimeLog timeLog) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();  // 执行原方法
        long cost = System.currentTimeMillis() - start;
        Log.d("AOP", joinPoint.getSignature().getName() + " 耗时: " + cost + "ms");
        return result;
    }
}

// 2. 自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LoginRequired {}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TimeLog {}

// 3. 使用注解（不需要改动业务逻辑）
public class OrderActivity extends AppCompatActivity {

    @LoginRequired  // AOP 自动插入登录检查
    public void placeOrder() {
        // 业务代码，不需要关心登录状态检查
    }

    @TimeLog  // AOP 自动记录耗时
    public void loadData() {
        // 加载数据
    }
}
```

#### 核心要点

```
AOP 使用场景：
├─ 登录检查（@LoginRequired）
├─ 权限检查（@RequiresPermission）
├─ 性能监控（@TimeLog，记录方法耗时）
├─ 行为埋点（@TrackEvent）
└─ 统一异常处理

Android 常见 AOP 应用：
├─ OkHttp 拦截器（责任链 = AOP 的一种实现）
├─ Retrofit + @Header（注解处理）
├─ Room @Dao（编译期生成代码）
└─ DataBinding（编译期生成 Binding 类）
```

---

### 15. 屏幕适配方案

#### 详细答案

Android 屏幕适配的核心是：**让同一套 UI 在不同尺寸/密度的设备上显示比例相同**。

| 方案 | 原理 | 优缺点 |
|------|------|-------|
| **dp** | 与密度无关的像素 | 基础方案，解决密度问题，无法解决尺寸问题 |
| **百分比布局** | wrap_content + 权重 | 简单，不精确 |
| **屏幕宽度限定符** | 不同 layout-sw360dp 目录 | 精确，维护成本高 |
| **今日头条方案** | 修改 density | 侵入性小，最流行 |
| **SmallestWidth** | sw限定符 + 自动生成 | 精确，编译期确定 |

#### 代码示例

```java
// 今日头条屏幕适配方案
// 核心：修改 DisplayMetrics.density，使 1dp = 设计稿 1dp 对应的像素

public class ScreenAdaptation {

    // 设计稿宽度（单位：dp）
    private static final float DESIGN_WIDTH_DP = 375f;  // 按 iPhone 6 设计稿

    public static void adaptScreen(Activity activity) {
        DisplayMetrics displayMetrics = activity.getResources().getDisplayMetrics();
        float screenWidth = displayMetrics.widthPixels;  // 屏幕宽度（px）

        // 计算适配后的 density：使屏幕宽度 = 设计稿宽度
        // density = 屏幕宽度(px) / 设计稿宽度(dp)
        float targetDensity = screenWidth / DESIGN_WIDTH_DP;

        displayMetrics.density = targetDensity;
        displayMetrics.densityDpi = (int) (targetDensity * 160);
        displayMetrics.scaledDensity = targetDensity;  // 字体大小也同步适配
    }

    // 取消适配（在 Dialog 等特殊场景恢复）
    public static void cancelAdapt(Activity activity) {
        float systemDensity = Resources.getSystem().getDisplayMetrics().density;
        DisplayMetrics displayMetrics = activity.getResources().getDisplayMetrics();
        displayMetrics.density = systemDensity;
        displayMetrics.densityDpi = (int) (systemDensity * 160);
    }
}

// 在 BaseActivity 中调用
public class BaseActivity extends AppCompatActivity {
    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        ScreenAdaptation.adaptScreen(this);  // 横竖屏切换时重新适配
    }

    @Override
    protected void onResume() {
        super.onResume();
        ScreenAdaptation.adaptScreen(this);
    }
}
```

```kotlin
// 关键单位换算
val density = Resources.getSystem().displayMetrics.density

// px → dp
fun px2dp(px: Int): Int = (px / density).toInt()

// dp → px
fun dp2px(dp: Int): Int = (dp * density).toInt()

// sp → px（字体）
fun sp2px(sp: Int): Int = TypedValue.applyDimension(
    TypedValue.COMPLEX_UNIT_SP, sp.toFloat(),
    Resources.getSystem().displayMetrics
).toInt()
```

#### 核心要点

```
屏幕相关概念：
├─ px：屏幕物理像素
├─ dp（dip）：与密度无关的单位，1dp = 1px（在 160dpi 的屏幕上）
├─ sp：字体单位，受系统字体大小设置影响
├─ density：density = dpi / 160（ldpi=0.75，mdpi=1，hdpi=1.5，xhdpi=2）
└─ densityDpi：每英寸像素数

今日头条方案原理：
└─ px = dp × density
   调整 density 使 px/density = 设计稿中的 dp 值
   让所有设备上，屏幕宽度 = 设计稿宽度（以 dp 计）

注意事项：
├─ 需要在 onResume 和 onConfigurationChanged 中重新设置
├─ 第三方 View 可能按系统 density 计算尺寸（需要恢复 density）
└─ 字体大小受系统字体设置影响（scaledDensity 需要特殊处理）
```

---

### 16. 64K 方法数问题

#### 详细答案

Android 早期的 `.dex` 文件格式用 16 位索引来引用方法，因此**单个 dex 文件最多只能引用 65536（64K）个方法**。超出则编译报错。

#### 代码示例

```groovy
// 1. 开启 MultiDex
android {
    defaultConfig {
        minSdkVersion 21  // API 21+ 原生支持多 dex，无需额外配置

        // API 21 以下需要：
        multiDexEnabled true
    }
}

dependencies {
    // API 21 以下需要 MultiDex 支持库
    implementation 'androidx.multidex:multidex:2.0.1'
}
```

```java
// 2. Application 中初始化（API 21 以下）
public class MyApplication extends MultiDexApplication {
    // 继承 MultiDexApplication 即可，无需其他操作
}

// 或者重写 attachBaseContext
public class MyApplication extends Application {
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);  // 在 super.onCreate() 之前安装
    }
}
```

#### 核心要点

```
64K 问题原因：
└─ dex 文件格式：方法引用用 16 位索引 → 最多 2^16 = 65536 个

解决方案：
├─ API 21+：ART 原生支持多 dex，直接加载（自动处理）
├─ API 21-：MultiDex 支持库，启动时动态加载额外 dex（第一次冷启动慢）

优化方向：
├─ 代码混淆（R8/ProGuard）减少方法数
├─ 删除无用依赖库
├─ 按需引入（如 Guava → 只引入需要的模块）
└─ minSdkVersion ≥ 21（公司业务允许时）
```

---

## 📌 Android 系统 SDK

### 17. ArrayMap 和 SparseArray 原理

#### 详细答案

两者都是为 **Android 场景优化**的替代 HashMap 的数据结构，适合**小数据量（< 1000）**的场景。

| 对比 | HashMap | ArrayMap | SparseArray |
|------|---------|---------|-------------|
| **实现** | 数组+链表 | 双数组 | 双数组 |
| **内存** | 高（Entry对象） | 低（无额外对象） | 最低 |
| **Key 类型** | 任意对象 | 任意对象 | int 只（避免装箱） |
| **扩容** | ×2（大步） | ×1.5（小步） | ×2 |
| **查找** | O(1) | O(log n) 二分 | O(log n) 二分 |
| **适用** | 大数据量 | 小数据量对象Key | 小数据量 int Key |

#### 代码示例

```java
// ArrayMap 使用（Key 是对象）
ArrayMap<String, User> userMap = new ArrayMap<>();
userMap.put("user_1", new User("张三"));
userMap.put("user_2", new User("李四"));
User user = userMap.get("user_1");

// SparseArray 使用（Key 只能是 int，避免 Integer 装箱）
SparseArray<User> sparseArray = new SparseArray<>();
sparseArray.put(1001, new User("张三"));  // Key 是 int，不装箱
sparseArray.put(1002, new User("李四"));
User user2 = sparseArray.get(1001);

// 其他变种：
// SparseBooleanArray：value 为 boolean
// SparseIntArray：value 为 int
// SparseLongArray：value 为 long
// LongSparseArray：key 为 long

// 选择建议
void chooseMap(int size) {
    if (size < 1000) {
        // key 是 int → SparseArray
        // key 是对象 → ArrayMap
    } else {
        // key 是 int 且 value 是 int → SparseIntArray
        // 其他 → HashMap
    }
}
```

#### 核心要点

```
ArrayMap 内部结构：
├─ int[] mHashes：存 key 的 hashCode（有序，支持二分查找）
└─ Object[] mArray：交替存 key 和 value（[k0,v0,k1,v1,...]）

SparseArray 内部结构：
├─ int[] mKeys：有序的 int Key 数组
└─ Object[] mValues：对应的 Value 数组
查找用二分查找 O(log n)

何时使用：
├─ 数据量 < 1000 条 → ArrayMap/SparseArray
├─ Key 是 int → SparseArray（避免 Integer 装箱）
└─ 数据量 > 1000 条 → HashMap（O(1) 优势更明显）
```

---

### 18. PathClassLoader 和 DexClassLoader 的区别

#### 详细答案

两者都继承自 `BaseDexClassLoader`，都能加载 `.dex` 和 `.apk`，区别在于**优化目录**。

| 对比 | PathClassLoader | DexClassLoader |
|------|----------------|---------------|
| **用途** | 加载已安装的 App | 加载任意路径的 dex/apk |
| **optimizedDirectory** | 固定（/data/dalvik-cache）| 可自定义 |
| **API 26+** | 两者无区别 | 两者无区别 |
| **应用** | Android 系统默认 | 热修复、插件化 |

#### 代码示例

```java
// DexClassLoader 加载外部 dex（热修复/插件化）
public class DexLoaderDemo {

    public void loadExternalDex(Context context, String dexPath) {
        // optimizedDirectory：dex 优化后的输出目录（API 26+ 忽略此参数）
        String optimizedDir = context.getCacheDir().getAbsolutePath();

        DexClassLoader loader = new DexClassLoader(
            dexPath,           // dex/apk 文件路径
            optimizedDir,      // 优化后 dex 的输出目录
            null,              // so 库路径
            context.getClassLoader()  // 父 ClassLoader
        );

        try {
            // 加载外部 dex 中的类
            Class<?> clazz = loader.loadClass("com.example.patch.FixedClass");
            Object instance = clazz.newInstance();
            // 通过反射调用方法
            Method method = clazz.getMethod("fixedMethod");
            method.invoke(instance);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

// Android 类加载器层级
/*
BootClassLoader（加载 Android 系统类）
    ↓ parent
PathClassLoader（加载 App 自身 dex）
    加载路径：/data/app/com.example/base.apk
*/
```

#### 核心要点

```
双亲委派模型：
ClassLoader 加载类时，先委托父 ClassLoader 加载
父 ClassLoader 找不到时，再由自己加载
→ 防止核心类被替换（如 java.lang.String）

插件化中使用 DexClassLoader：
└─ 为插件 apk 创建独立的 DexClassLoader
   插件中的类由该 ClassLoader 加载
   通过反射/接口调用插件功能
```

---

### 19. Lifecycle 的工作原理

#### 详细答案

`Lifecycle` 是 Jetpack 组件，让自定义组件能**感知 Activity/Fragment 的生命周期**，避免手动注册/注销。

**核心机制（API 29 之前）：**
向 Activity 注入一个无 UI 的 Fragment（`ReportFragment`），通过 Fragment 的生命周期回调来触发 Lifecycle 事件。

**API 29+ 机制：**
使用 `Activity.registerActivityLifecycleCallbacks()` 直接监听。

#### 代码示例

```kotlin
// 1. 自定义 LifecycleObserver
class MyLocationManager(private val lifecycle: Lifecycle) : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        startLocationUpdates()  // Activity onStart 时开启定位
    }

    override fun onStop(owner: LifecycleOwner) {
        stopLocationUpdates()   // Activity onStop 时停止定位
    }

    fun observeLocation(owner: LifecycleOwner) {
        lifecycle.addObserver(this)
    }
}

// 在 Activity 中使用
class MapActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 不需要在 onStart/onStop 中手动管理
        MyLocationManager(lifecycle).observeLocation(this)
    }
}

// 2. 自定义组件感知生命周期
class VideoPlayer(private val lifecycle: Lifecycle) : LifecycleEventObserver {

    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        when (event) {
            Lifecycle.Event.ON_RESUME -> resume()
            Lifecycle.Event.ON_PAUSE  -> pause()
            Lifecycle.Event.ON_DESTROY -> {
                release()
                lifecycle.removeObserver(this)
            }
            else -> {}
        }
    }

    init {
        lifecycle.addObserver(this)
    }
}

// 3. 检查当前生命周期状态
fun doSomethingIfActive(lifecycleOwner: LifecycleOwner) {
    if (lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.STARTED)) {
        // Activity 处于 STARTED 或更高状态才执行
    }
}
```

#### 核心要点

```
Lifecycle 核心类：
├─ Lifecycle：抽象类，持有生命周期状态（State）和观察者列表
├─ LifecycleOwner：拥有 Lifecycle 的组件（Activity/Fragment）
├─ LifecycleObserver：观察者接口
└─ LifecycleRegistry：Lifecycle 的具体实现

实现原理（API 29 之前）：
└─ ComponentActivity.onCreate() 中注入 ReportFragment
   ReportFragment 的生命周期方法触发 LifecycleRegistry.handleLifecycleEvent()

Lifecycle.State：
DESTROYED < INITIALIZED < CREATED < STARTED < RESUMED
```

---

## 📌 Kotlin 补充

### 20. Kotlin Sequence（序列）

#### 详细答案

`Sequence` 是**惰性**（懒计算）的集合操作，只在终端操作（`toList()`/`first()`）被调用时才实际执行。

**与普通集合操作的区别：**
- **普通集合**：每个操作（map/filter）都创建中间集合，水平处理
- **Sequence**：纵向处理（每个元素走完所有操作），不创建中间集合

#### 代码示例

```kotlin
// 1. 普通集合 vs Sequence
fun collectionVsSequence() {
    val list = (1..1_000_000).toList()

    // 普通集合：创建 3 个中间集合（map结果 + filter结果 + take结果）
    val result1 = list
        .map { it * 2 }      // 创建 100万 个元素的新集合
        .filter { it > 10 }  // 创建另一个新集合
        .take(5)             // 再创建一个

    // Sequence：惰性，取到 5 个就停止，不处理剩余元素
    val result2 = list.asSequence()
        .map { it * 2 }      // 不立即执行
        .filter { it > 10 }  // 不立即执行
        .take(5)             // 不立即执行
        .toList()            // 终端操作，此时才真正执行（只处理前几个元素）
}

// 2. 无限序列
fun infiniteSequence() {
    // generateSequence：生成无限序列
    val naturals = generateSequence(1) { it + 1 }  // 1, 2, 3, ...

    val first10Evens = naturals
        .filter { it % 2 == 0 }
        .take(10)  // 只取前 10 个偶数
        .toList()  // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

    // 斐波那契数列
    val fibonacci = generateSequence(Pair(0, 1)) { (a, b) -> Pair(b, a + b) }
        .map { it.first }
    println(fibonacci.take(10).toList())  // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
}

// 3. sequence builder（自定义序列）
fun treeSequence(root: TreeNode?): Sequence<Int> = sequence {
    if (root == null) return@sequence
    yieldAll(treeSequence(root.left))  // 惰性递归
    yield(root.value)                  // 产出当前值
    yieldAll(treeSequence(root.right))
}
```

#### 核心要点

```
何时用 Sequence？
├─ 数据量大（> 数千）→ Sequence 避免中间集合的内存压力
├─ 只需要部分结果（配合 take/first）→ 及早退出，不处理所有元素
└─ 数据量小 → 普通集合（函数调用开销更小）

Sequence 操作分类：
├─ 中间操作（惰性）：map、filter、take、drop、flatMap
└─ 终端操作（触发执行）：toList、toSet、first、count、sum、forEach

Sequence vs Java Stream：
├─ Kotlin Sequence：单线程惰性序列
└─ Java Stream：支持并行（parallelStream），功能更丰富
```

---

### 21. Kotlin infix 函数

#### 详细答案

`infix` 函数允许以**中缀表达式**的方式调用，使代码更接近自然语言。

**条件：**
1. 必须是成员函数或扩展函数
2. 只能有一个参数
3. 参数不能有默认值，不能是可变参数

#### 代码示例

```kotlin
// 1. 内置 infix 函数
fun inbuiltInfix() {
    // to（创建 Pair）
    val pair = "key" to "value"  // 等同于 "key".to("value")

    val map = mapOf("a" to 1, "b" to 2)

    // until（范围）
    for (i in 1 until 10) print(i)  // 1到9

    // step
    for (i in 0..20 step 4) print(i)  // 0,4,8,12,16,20

    // and/or/xor（位运算）
    val result = 0b1010 and 0b1100  // 0b1000
}

// 2. 自定义 infix 函数
// DSL 风格的单元测试断言
infix fun <T> T.shouldBe(expected: T) {
    if (this != expected) throw AssertionError("期望: $expected, 实际: $this")
}

infix fun <T> T.shouldNotBe(expected: T) {
    if (this == expected) throw AssertionError("不应该等于: $expected")
}

// 使用
fun testSum() {
    val result = 2 + 2
    result shouldBe 4
    result shouldNotBe 5
}

// 3. 中缀函数用于 DSL
class HttpRequest {
    var method = "GET"
    var url = ""
    val headers = mutableMapOf<String, String>()
}

infix fun HttpRequest.GET(url: String): HttpRequest {
    this.method = "GET"
    this.url = url
    return this
}

infix fun HttpRequest.header(pair: Pair<String, String>): HttpRequest {
    headers[pair.first] = pair.second
    return this
}

// 使用（接近自然语言）
val request = HttpRequest() GET "https://api.example.com" header ("Authorization" to "Bearer token")
```

#### 核心要点

```
infix 函数特点：
├─ 语法糖：a infix b 等同于 a.infix(b)
├─ 提升可读性（尤其在 DSL 和测试断言中）
└─ 优先级低于算术运算符，高于比较运算符（>、<、==）

常见内置 infix 函数：
├─ to：创建 Pair（Pair("key", "value")）
├─ until：不含终点的范围（1 until 10 = 1..9）
├─ downTo：递减范围（10 downTo 1）
├─ step：步长（1..10 step 2）
└─ and/or/xor：位运算
```

---

### 22. Kotlin 构造方法详解

#### 代码示例

```kotlin
// 1. 主构造函数（Primary Constructor）
class User(
    val id: Long,           // val/var 直接声明属性
    val name: String,
    private val email: String,
    val age: Int = 18       // 默认参数
) {
    // init 块在主构造函数执行后立即执行
    init {
        require(name.isNotBlank()) { "名字不能为空" }
        require(age > 0) { "年龄必须大于0" }
    }
}

// 2. 次构造函数（Secondary Constructor）
// 必须直接或间接委托给主构造函数
class Order(val id: Long, val items: List<String>) {

    var totalPrice: Double = 0.0

    // 次构造函数（委托给主构造函数）
    constructor(id: Long) : this(id, emptyList()) {
        // 次构造函数体在主构造函数和 init 块之后执行
    }

    constructor(id: Long, vararg items: String) : this(id, items.toList())
}

// 3. 工厂方法替代构造函数（推荐）
class Database private constructor(val url: String) {
    companion object {
        fun create(url: String): Database {
            require(url.startsWith("jdbc:")) { "无效的数据库 URL" }
            return Database(url)
        }

        // 多个工厂方法，语义清晰
        fun createMemory() = Database("jdbc:h2:mem:")
        fun createFile(path: String) = Database("jdbc:h2:$path")
    }
}

// 4. 构造函数执行顺序
class Parent(val name: String) {
    init { println("Parent init: $name") }
}

class Child(name: String, val age: Int) : Parent(name) {
    init { println("Child init: $name, $age") }

    constructor(name: String) : this(name, 0) {
        println("Child 次构造函数")
    }
}

// new Child("Tom") 输出顺序：
// 1. Parent init: Tom
// 2. Child init: Tom, 0
// 3. Child 次构造函数
```

#### 核心要点

```
构造函数执行顺序：
父类主构造 → 父类 init → 子类主构造 → 子类 init → 子类次构造

主构造 vs 次构造：
├─ 主构造：简洁，在类头部声明，适合大多数场景
├─ 次构造：灵活，可以有更复杂的逻辑
└─ 次构造必须委托给主构造（直接 or 间接）

最佳实践：
├─ 优先使用主构造 + 默认参数
├─ 需要工厂方法时，私有化构造函数 + companion object
└─ 避免大量次构造函数（用默认参数代替）
```

---

### 23. Kotlin Any 与 Java Object 的区别

#### 代码示例

```kotlin
// 1. Any 是 Kotlin 所有类的根类（非空）
fun anyDemo() {
    val obj: Any = "hello"
    val num: Any = 42
    val list: Any = listOf(1, 2, 3)

    // Any 只有三个方法
    obj.equals(num)    // 比较
    obj.hashCode()     // 哈希码
    obj.toString()     // 字符串表示
}

// 2. Any vs Object 的区别
fun differences() {
    // Kotlin Any → 编译为 Java Object（非空）
    // Kotlin Any? → 可空的 Object

    // Java Object 在 Kotlin 中是 Any?（平台类型）
    val javaObj: Any = Object()  // Kotlin Any

    // wait/notify/getClass 等 Java Object 方法
    // 在 Kotlin 中不能直接通过 Any 调用
    // 需要显式转换为 java.lang.Object
    val jObj = javaObj as java.lang.Object
    // jObj.wait()  // 可以调用 Java Object 的方法
}

// 3. 类型检查
fun typeCheck(obj: Any) {
    when (obj) {
        is String -> println("字符串: ${obj.length}")   // 智能转换
        is Int    -> println("整数: ${obj + 1}")
        is List<*> -> println("列表: ${obj.size}")
        else      -> println("其他: $obj")
    }
}

// 4. Kotlin 数据类型与 Java 对比
/*
Kotlin          Java（编译后）
Any       →   Object
Any?      →   @Nullable Object
Int       →   int（基本类型，自动优化）
Int?      →   Integer（装箱类型）
Unit      →   void（返回值）/ Unit（其他场景）
Nothing   →   没有对应（函数不会返回，如 throw）
*/
```

#### 核心要点

```
Any 与 Object 区别：
├─ Any 不包含 wait/notify/getClass 等线程方法
├─ Any 是非空类型，Any? 才对应可空 Object
├─ Kotlin 原始类型（Int/Boolean）不继承 Any（自动优化为 Java 基本类型）
└─ Kotlin 在 JVM 上 Any 编译为 Object

Kotlin 特殊类型：
├─ Unit：类似 void，但是真实类型（可以作为泛型参数）
├─ Nothing：函数永远不会正常返回（throw/无限循环）
└─ Nothing?：唯一的值是 null
```

---

## 📌 其他补充

### 24. TCP/IP 分层模型

#### 详细答案

| 层 | TCP/IP 四层 | OSI 七层 | 协议 |
|---|------------|---------|------|
| 4 | 应用层 | 应用层/表示层/会话层 | HTTP、HTTPS、FTP、DNS |
| 3 | 传输层 | 传输层 | TCP、UDP |
| 2 | 网络层 | 网络层 | IP、ICMP、ARP |
| 1 | 网络接口层 | 数据链路层/物理层 | Ethernet、Wi-Fi |

**HTTP 属于应用层，TCP 属于传输层。**

#### 代码示例

```java
// TCP vs UDP 对比
public class TCPvsUDP {
    /*
    TCP（Transmission Control Protocol）：
    ├─ 面向连接（三次握手建立连接）
    ├─ 可靠传输（确认重传、流量控制、拥塞控制）
    ├─ 有序（保证数据顺序）
    ├─ 速度较慢
    └─ 适用：HTTP/HTTPS、FTP、邮件（对可靠性要求高）

    UDP（User Datagram Protocol）：
    ├─ 无连接（直接发送数据报）
    ├─ 不可靠（可能丢包、乱序）
    ├─ 速度快（无握手和确认开销）
    └─ 适用：直播/视频通话（允许丢帧，对延迟敏感）、DNS查询
    */

    // Android UDP Socket
    public void udpDemo() throws Exception {
        DatagramSocket socket = new DatagramSocket();
        byte[] data = "Hello".getBytes();
        DatagramPacket packet = new DatagramPacket(
            data, data.length,
            InetAddress.getByName("192.168.1.1"), 8080
        );
        socket.send(packet);
        socket.close();
    }
}
```

---

### 25. 面向对象六大原则（SOLID）

#### 详细答案

| 原则 | 缩写 | 含义 |
|------|------|------|
| 单一职责原则 | SRP | 一个类只做一件事 |
| 开闭原则 | OCP | 对扩展开放，对修改关闭 |
| 里氏替换原则 | LSP | 子类可以替换父类 |
| 接口隔离原则 | ISP | 接口应小而专 |
| 依赖倒置原则 | DIP | 依赖抽象而非具体实现 |
| 迪米特法则 | LoD | 只与直接朋友通信 |

#### 代码示例

```kotlin
// 1. 单一职责（SRP）
// ❌ 违反：User 既处理用户数据，又处理数据库，又处理邮件
class BadUser {
    fun saveToDatabase() {}
    fun sendEmail() {}
    fun formatForDisplay(): String = ""
}

// ✅ 遵守：每个类只负责一件事
class User(val name: String, val email: String)
class UserRepository { fun save(user: User) {} }
class EmailService { fun send(email: String, content: String) {} }

// 2. 开闭原则（OCP）
// ❌ 违反：新增支付方式需要修改 PaymentService
class BadPaymentService {
    fun pay(type: String, amount: Double) {
        when (type) {
            "alipay" -> payWithAlipay(amount)
            "wechat" -> payWithWechat(amount)
            // 新增支付方式需要修改此处 ❌
        }
    }
}

// ✅ 遵守：通过接口扩展，不修改已有代码
interface PaymentStrategy {
    fun pay(amount: Double)
}
class AlipayStrategy : PaymentStrategy { override fun pay(amount: Double) {} }
class WechatPayStrategy : PaymentStrategy { override fun pay(amount: Double) {} }
class PaymentService(private val strategy: PaymentStrategy) {
    fun pay(amount: Double) = strategy.pay(amount)
    // 新增支付方式只需新增 Strategy 类，不修改此类 ✅
}

// 3. 依赖倒置（DIP）
// ❌ 高层依赖低层实现
class BadOrderService {
    private val db = MySQLDatabase()  // 直接依赖具体实现
}

// ✅ 高层依赖抽象（接口）
interface Database { fun save(data: Any) }
class MySQLDatabase : Database { override fun save(data: Any) {} }
class OrderService(private val db: Database) {  // 依赖抽象
    fun createOrder(order: Any) = db.save(order)
}
```

#### 核心要点

```
六大原则记忆：
SRP → 一个类一个责任（类不要臃肿）
OCP → 加功能加代码，不改已有代码（用接口/抽象扩展）
LSP → 子类不破坏父类契约（Override 要符合父类语义）
ISP → 接口要小，不强迫实现不需要的方法
DIP → 依赖接口，不依赖实现（便于测试/替换）
LoD → 减少耦合，不了解不调用
```

---

### 26. 原型模式（Prototype）

#### 详细答案

原型模式通过**复制已有对象**来创建新对象，而不是通过 `new`。适用于**创建对象成本高**，或需要**大量相似对象**的场景。

#### 代码示例

```java
// 1. 实现 Cloneable（浅拷贝）
public class UserProfile implements Cloneable {
    private String name;
    private int age;
    private List<String> skills;  // 引用类型

    @Override
    public UserProfile clone() {
        try {
            return (UserProfile) super.clone();  // 浅拷贝（skills 是同一个引用）
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

// 2. 深拷贝
public class DeepCopyUserProfile implements Cloneable {
    private String name;
    private List<String> skills;

    @Override
    public DeepCopyUserProfile clone() {
        try {
            DeepCopyUserProfile copy = (DeepCopyUserProfile) super.clone();
            copy.skills = new ArrayList<>(this.skills);  // 深拷贝 List
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

```kotlin
// Kotlin data class 的 copy 就是原型模式
data class Config(
    val host: String = "localhost",
    val port: Int = 8080,
    val debug: Boolean = false
)

fun prototypeDemo() {
    val defaultConfig = Config()
    val devConfig = defaultConfig.copy(debug = true)         // 原型模式
    val prodConfig = defaultConfig.copy(host = "prod.server.com")

    // Android 中 Bundle 的使用也类似原型模式
    val bundle = Bundle()
    bundle.putString("key", "value")
    val newBundle = bundle.clone() as Bundle  // 复制 Bundle
}
```

#### 核心要点

```
原型模式 vs new：
├─ new：每次重新初始化（成本高）
└─ clone：基于已有对象复制（成本低）

浅拷贝 vs 深拷贝：
├─ 浅拷贝：引用类型字段共享（修改一个影响另一个）
└─ 深拷贝：引用类型字段也复制（完全独立）

Android 中的原型模式：
├─ Bundle.clone()
├─ Intent 的 putExtras()
└─ Kotlin data class 的 copy()
```

---

## 📊 Part 5 完成总结

### ✅ 已完成（32 题）

**Android UI 绘制补充（9 题）：**
1. ✅ RecyclerView 四级缓存机制
2. ✅ Fragment 生命周期详解
3. ✅ SurfaceView vs TextureView
4. ✅ LayoutInflater.inflate 原理
5. ✅ View.post() 获取宽高原理
6. ✅ 同步屏障（MessageQueue）
7. ✅ Fragment 懒加载（ViewPager/ViewPager2）
8. ✅ ViewDragHelper 工作原理
9. ✅ RecyclerView.Adapter 刷新方式（DiffUtil）

**第三方框架（3 题）：**
10. ✅ EventBus 实现原理（注解+反射+APT）
11. ✅ LeakCanary 工作原理（WeakReference+ReferenceQueue）
12. ✅ RxJava 背压原理（Flowable vs Observable）

**综合技术（4 题）：**
13. ✅ 热修复原理（Dex 插桩 / 方法替换）
14. ✅ AOP 面向切面编程（AspectJ）
15. ✅ 屏幕适配（今日头条方案）
16. ✅ 64K 方法数问题（MultiDex）

**Android 系统 SDK（3 题）：**
17. ✅ ArrayMap 和 SparseArray 原理
18. ✅ PathClassLoader vs DexClassLoader
19. ✅ Lifecycle 工作原理

**Kotlin 补充（4 题）：**
20. ✅ Kotlin Sequence（惰性序列）
21. ✅ infix 关键字
22. ✅ 构造方法详解（主/次/init执行顺序）
23. ✅ Any vs Object

**其他（3 题）：**
24. ✅ TCP/IP 分层（四层 vs OSI 七层）
25. ✅ 面向对象六大原则（SOLID）
26. ✅ 原型模式（浅拷贝/深拷贝）

---

## 📚 五部分完整汇总

| Part | 主题 | 题数 |
|------|------|------|
| **Part 1** | 四大组件 + 异步消息机制 | 22 题 |
| **Part 2** | Android UI 绘制 + 性能优化 | 45 题 |
| **Part 3** | 架构设计 + 网络 + 存储 + IPC | 35 题 |
| **Part 4** | Kotlin + 数据结构 + 设计模式 | 40 题 |
| **Part 5** | UI补充 + 框架原理 + 综合技术 | 32 题 |
| **🎉 合计** | | **174 题** |

---

**恭喜！Android 面试题库已覆盖 174 道核心题目！** 🎉

建议配合快速复习卡（Java_Quick_Review_Cards.md）一起使用效果更佳。