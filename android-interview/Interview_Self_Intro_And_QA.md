# Android 面试备考手册

> 基于个人简历定制，覆盖自我介绍 + 第一/二优先级高频考点
> 更新日期：2026-03-05

---

## 一、自我介绍（60-90秒版本）

> 建议背熟后自然说出来，不要背稿感太强。

"面试官好，我叫何某某，有 8 年 Android 开发经验。

技术上，我熟悉 MVVM/MVP 等主流架构，熟练使用 Kotlin 协程和 Flow 做响应式开发，对 View 绘制、事件分发、性能优化有较深入的理解。除了应用层开发，我还有嵌入式 Android 硬件开发经验，做过 GPIO、串口、蓝牙等底层通信，也有 Flutter 跨端和 Web3 区块链的项目经历。

最近在 YLS 这家公司，我负责 Android 技术体系建设与团队管理，主导了一款健康穿戴类 APP 的开发——接入 Google Health Connect，整合蓝牙戒指设备数据，实现步数、心率、睡眠等指标同步，同时集成了 AI 健康助手功能。项目里我引入了 Claude Code 配合 MCP 工具，通过编写 CLAUDE.md 规则文件，让 AI 生成的代码自动对齐项目技术栈，显著提升了团队研发效率。

我期望找一个能持续深耕 Android 技术、有一定技术挑战的岗位，深圳优先。以上是我的简单介绍，谢谢。"

---

### 自我介绍结构拆解

| 段落 | 内容 | 时长 |
|------|------|------|
| 开场 | 名字 + 年限 | 5秒 |
| 技术广度 | 架构 + 协程 + 硬件 + Flutter + Web3 | 20秒 |
| 近期亮点项目 | YLS 健康 APP + AI 工具落地 | 35秒 |
| 求职意向 | 技术挑战 + 深圳 | 10秒 |

---

# 第一优先级：面试题详细答案

---

## 模块一：Kotlin 协程与 Flow

---

### Q1：协程和线程有什么区别？

**答：**

| 对比项 | 线程 | 协程 |
|--------|------|------|
| 开销 | 每个线程约 1MB 内存 | 极轻量，可创建数万个 |
| 切换 | 系统内核调度，有上下文切换开销 | 用户态挂起/恢复，无内核切换 |
| 阻塞 | 阻塞线程，资源浪费 | 挂起不阻塞线程，线程可做其他事 |
| 编程模型 | 回调嵌套，代码复杂 | 顺序书写异步代码，可读性高 |

**核心理解：** 协程是运行在线程上的轻量级任务，多个协程可以共享同一个线程，通过"挂起（suspend）"和"恢复（resume）"来实现并发，不需要创建大量线程。

```kotlin
// ============================================================
// 场景：先请求用户信息，再请求订单列表，最后更新 UI
// ============================================================

// ❌ 线程写法 —— 回调地狱，层层嵌套，难以维护
fun loadDataByThread() {
    Thread {
        val user = api.fetchUser()          // 阻塞当前线程等待结果
        runOnUiThread {                     // 切回主线程
            showUser(user)
            Thread {
                val orders = api.fetchOrders(user.id)  // 又阻塞线程
                runOnUiThread {
                    showOrders(orders)      // 再切回主线程
                    // 如果再加一层请求，嵌套会更深……
                }
            }.start()
        }
    }.start()
}

// ✅ 协程写法 —— 顺序书写，逻辑清晰，和同步代码一样好读
fun loadDataByCoroutine() {
    lifecycleScope.launch {
        // 自动在 IO 线程执行网络请求，挂起期间主线程不被阻塞
        val user = withContext(Dispatchers.IO) { api.fetchUser() }
        showUser(user)                      // 自动回到主线程更新 UI

        val orders = withContext(Dispatchers.IO) { api.fetchOrders(user.id) }
        showOrders(orders)                  // 自动回到主线程更新 UI
    }
}

// ✅ 进阶：两个请求并行执行，更高效
fun loadDataParallel() {
    lifecycleScope.launch {
        // async 启动两个并行协程，同时发起请求
        val userDeferred  = async(Dispatchers.IO) { api.fetchUser() }
        val configDeferred = async(Dispatchers.IO) { api.fetchConfig() }

        // await() 等待各自结果，谁慢等谁，但两个请求是同时进行的
        val user   = userDeferred.await()
        val config = configDeferred.await()

        // 两个都拿到后，更新 UI
        showUserAndConfig(user, config)
    }
}
```

---

### Q2：CoroutineScope、Job、Dispatcher 各是什么关系？

**答：**

- **CoroutineScope**：协程的"容器"，定义协程的生命周期范围，`launch`/`async` 必须在某个 Scope 里调用
- **Job**：协程的"控制句柄"，可以取消、等待、查状态
- **Dispatcher**：协程的"调度器"，决定协程运行在哪个线程

**关系类比：** Scope 是工厂（管生命周期）→ Job 是工人（每个任务的引用）→ Dispatcher 是工位（线程）

```kotlin
// ============================================================
// Dispatcher 四种类型详解
// ============================================================
Dispatchers.Main        // UI 主线程，只能做 UI 操作，不能做耗时任务
Dispatchers.IO          // IO 线程池（最多64个线程），适合网络/文件/数据库
Dispatchers.Default     // CPU 线程池（线程数 = CPU核心数），适合复杂计算/排序/JSON解析
Dispatchers.Unconfined  // 不指定线程，在哪个线程挂起就在哪个线程恢复（慎用）


// ============================================================
// Job 的使用：控制协程的取消、等待
// ============================================================
class MyViewModel : ViewModel() {

    // viewModelScope 是 ViewModel 自带的 Scope
    // 当 ViewModel 销毁时，它内部所有协程自动取消
    fun loadData() {
        // launch 返回一个 Job，可以用来手动取消
        val job: Job = viewModelScope.launch(Dispatchers.IO) {
            val result = repository.fetchData()         // IO 线程执行
            withContext(Dispatchers.Main) {             // 切换到主线程
                _uiState.value = result
            }
        }

        // 如果需要提前取消这个任务（比如用户切换 Tab）
        // job.cancel()

        // 如果需要等待任务完成（在另一个协程中）
        // job.join()

        // 查看状态
        // job.isActive    → 是否还在运行
        // job.isCancelled → 是否已取消
        // job.isCompleted → 是否已完成
    }
}


// ============================================================
// 自定义 Scope：Activity 中手动管理（了解即可，实际用 lifecycleScope）
// ============================================================
class MyActivity : AppCompatActivity() {

    // 创建一个和 Activity 生命周期绑定的 Scope
    private val myScope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    override fun onDestroy() {
        super.onDestroy()
        myScope.cancel()  // 手动取消，防止内存泄漏
    }
}
// 实际开发直接用 lifecycleScope（Activity/Fragment）或 viewModelScope（ViewModel）
// 它们会自动在合适的时机取消，不需要手动管理
```

---

### Q3：Flow、StateFlow、SharedFlow 区别？

**答：**

| 类型 | 冷/热 | 特点 | 典型用途 |
|------|-------|------|----------|
| Flow | **冷流** | 每次 `collect` 都重新执行，无订阅者时不产生数据 | 网络请求、数据库查询 |
| StateFlow | **热流** | 始终保留最新值，新订阅者立即收到当前值 | UI 状态（loading/success/error）|
| SharedFlow | **热流** | 可配置回放数量，适合广播事件 | 一次性事件（Toast、导航跳转）|

```kotlin
// ============================================================
// 1. 普通 Flow —— 冷流，collect 才开始执行
// ============================================================

// 每次有人 collect，flow 块就重新执行一遍
fun getUserFlow(id: Int): Flow<User> = flow {
    println("开始请求...")          // 只有 collect 时才会打印
    val user = api.fetchUser(id)    // 执行网络请求
    emit(user)                      // 把结果发送给收集方
}

// 使用时
lifecycleScope.launch {
    getUserFlow(123)
        .map { it.name }            // 转换：取名字
        .filter { it.isNotEmpty() } // 过滤：过滤空名字
        .collect { name ->
            textView.text = name
        }
}


// ============================================================
// 2. StateFlow —— 热流，专门用于 UI 状态
// ============================================================
class UserViewModel : ViewModel() {

    // _uiState 是可写的（私有），uiState 是只读的（对外暴露）
    // 初始值必须给，这里给 Loading
    private val _uiState = MutableStateFlow<UiState<User>>(UiState.Loading)
    val uiState: StateFlow<UiState<User>> = _uiState.asStateFlow()

    fun loadUser(id: Int) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading          // 1. 先设置为加载中
            try {
                val user = withContext(Dispatchers.IO) {
                    repository.getUser(id)
                }
                _uiState.value = UiState.Success(user) // 2. 成功，更新数据
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "请求失败") // 3. 失败，更新错误
            }
        }
    }
}

// Activity 中收集 StateFlow
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            // repeatOnLifecycle：App 退后台时自动暂停收集，回到前台自动恢复
            // 这样不会在后台浪费资源
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UiState.Loading      -> showLoading()
                        is UiState.Success<User> -> showUser(state.data)
                        is UiState.Error        -> showError(state.message)
                    }
                }
            }
        }
    }
}


// ============================================================
// 3. SharedFlow —— 热流，专门用于一次性事件（事件只消费一次）
// ============================================================
class LoginViewModel : ViewModel() {

    // replay = 0 表示新订阅者不会收到历史事件（一次性事件不应该重放）
    private val _events = MutableSharedFlow<LoginEvent>(replay = 0)
    val events: SharedFlow<LoginEvent> = _events.asSharedFlow()

    fun login(username: String, password: String) {
        viewModelScope.launch {
            val success = repository.login(username, password)
            if (success) {
                _events.emit(LoginEvent.NavigateToHome)   // 发送导航事件
            } else {
                _events.emit(LoginEvent.ShowToast("登录失败，请检查密码"))
            }
        }
    }
}

// 事件类型定义
sealed class LoginEvent {
    object NavigateToHome : LoginEvent()
    data class ShowToast(val message: String) : LoginEvent()
}

// Activity 中收集 SharedFlow 事件
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is LoginEvent.NavigateToHome  -> startActivity(Intent(this, HomeActivity::class.java))
                is LoginEvent.ShowToast       -> Toast.makeText(this, event.message, Toast.LENGTH_SHORT).show()
            }
        }
    }
}

// 关键记忆口诀：
// StateFlow → 状态（有初始值，保留最新，随时可读）
// SharedFlow → 事件（无初始值，一次性，不重放）
// Flow → 数据源（冷，按需执行）
```

---

### Q4：协程取消与结构化并发是什么？

**答：**

**结构化并发：** 子协程的生命周期由父协程管理，父取消 → 所有子自动取消，不会泄漏协程。

```kotlin
// ============================================================
// 结构化并发：父取消，子全部取消
// ============================================================
class MyViewModel : ViewModel() {

    fun loadPageData() {
        // viewModelScope 是父 Scope
        viewModelScope.launch {
            // job1 和 job2 都是这个协程的子协程
            val job1 = launch(Dispatchers.IO) {
                fetchUserProfile()  // 子协程 1
            }
            val job2 = launch(Dispatchers.IO) {
                fetchBannerList()   // 子协程 2
            }

            job1.join()  // 等待 job1 完成
            job2.join()  // 等待 job2 完成

            // 当 ViewModel 销毁时，viewModelScope 自动取消
            // job1 和 job2 也会自动取消，不需要手动管理
        }
    }
}


// ============================================================
// 协程取消的几个重要细节
// ============================================================

// 细节1：delay() 是可取消的，但普通的 Thread.sleep() 不是
suspend fun goodSuspendFun() {
    delay(1000)        // ✅ 协程被取消时，delay 会立刻抛出 CancellationException
}

fun badBlockingFun() {
    Thread.sleep(1000) // ❌ 协程被取消时，这行不响应取消，会继续阻塞线程
}


// 细节2：CPU 密集型循环需要手动检查取消状态
suspend fun processLargeData(list: List<Int>): List<Int> {
    val result = mutableListOf<Int>()
    for (item in list) {
        ensureActive()          // ← 关键！每次循环检查协程是否被取消
                                // 如果已取消，抛出 CancellationException，停止循环
        result.add(heavyProcess(item))
    }
    return result
}


// 细节3：finally 块在协程取消后依然会执行（保证资源释放）
suspend fun openAndProcessFile(path: String) {
    val file = openFile(path)
    try {
        processFile(file)       // 如果这里挂起时协程被取消…
    } finally {
        file.close()            // ← 依然会执行！保证文件被关闭
    }
}


// 细节4：SupervisorJob —— 子协程失败不影响兄弟协程
// 普通 Job：任何子协程失败，父和所有兄弟都取消
// SupervisorJob：某个子协程失败，其他子协程继续运行
class HomeViewModel : ViewModel() {

    fun loadHomePage() {
        // viewModelScope 内部使用 SupervisorJob，所以互相独立
        viewModelScope.launch {
            // launch1 失败不影响 launch2
            launch { fetchBanner() }
            launch { fetchRecommend() }
            launch { fetchUserInfo() }
        }
    }
}
```

---

## 模块二：MVVM 架构

---

### Q5：ViewModel 为什么在屏幕旋转后不会被销毁？

**答：**

ViewModel 存储在 `ViewModelStore` 中。屏幕旋转时 Activity 重建，但系统通过 `NonConfigurationInstances` 机制把 `ViewModelStore` 保留下来，传递给新的 Activity，所以 ViewModel 实例不变。

```kotlin
// ============================================================
// ViewModel 生命周期完整示意
// ============================================================

// Activity 旋转时的生命周期：
// onCreate → onStart → onResume
//    [用户旋转屏幕]
// onPause → onStop → onDestroy（旧 Activity 销毁）
// onCreate → onStart → onResume（新 Activity 创建）

// ViewModel 生命周期：
// ViewModel.init()  ← 第一次 onCreate 时创建
//    [Activity 旋转、配置变更 —— ViewModel 完全不受影响]
// ViewModel.onCleared()  ← 只在 Activity 真正 finish 时才调用

// ============================================================
// 实际代码示例：旋转不丢失数据
// ============================================================
class CounterViewModel : ViewModel() {
    // 这个数据在旋转后依然保留！
    var count = mutableStateOf(0)
        private set

    fun increment() { count.value++ }
}

class CounterActivity : AppCompatActivity() {
    // by viewModels() 保证同一个 Activity 拿到同一个 ViewModel 实例
    private val viewModel: CounterViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 旋转后 onCreate 再次调用，但 viewModel.count 依然是旋转前的值
        println("当前计数：${viewModel.count.value}")
    }
}


// ============================================================
// ViewModel 真正销毁的时机
// ============================================================

// 销毁场景1：用户按返回键
// 销毁场景2：调用 finish()
// 销毁场景3：系统 kill 进程

// 不销毁场景（配置变更）：
// - 屏幕旋转
// - 深色/浅色模式切换
// - 语言切换
// - 字体大小变化
// 这些情况 Activity 重建，但 ViewModel 都不会销毁


// ============================================================
// onCleared() —— ViewModel 销毁前的清理工作
// ============================================================
class UserViewModel : ViewModel() {
    private val disposables = CompositeDisposable()  // RxJava

    override fun onCleared() {
        super.onCleared()
        disposables.clear()  // 在这里清理 RxJava 订阅，防止内存泄漏
        // 注意：协程用 viewModelScope 不需要手动清理，会自动取消
    }
}
```

---

### Q6：LiveData 和 StateFlow 如何选择？

**答：**

| 特性 | LiveData | StateFlow |
|------|----------|-----------|
| 生命周期感知 | 自动感知，无需手动取消 | 需配合 `repeatOnLifecycle` |
| 初始值 | 无（可能为 null）| 必须有初始值 |
| 线程安全 | `postValue` 可从任意线程调用 | 需在协程中 emit |
| 转换操作符 | 只有 `map`/`switchMap` | 丰富：`map`/`filter`/`combine`/`zip` 等 |
| 与 Flow 互操作 | `asFlow()` 转换 | 天然就是 Flow |

**推荐：新项目全用 StateFlow**，配合 `repeatOnLifecycle` 安全收集

```kotlin
// ============================================================
// 对比1：LiveData 写法
// ============================================================
class OldViewModel : ViewModel() {
    private val _userName = MutableLiveData<String>()
    val userName: LiveData<String> = _userName

    fun loadUser() {
        viewModelScope.launch(Dispatchers.IO) {
            val user = repository.getUser()
            _userName.postValue(user.name)  // postValue 线程安全
        }
    }
}

// Activity 观察 LiveData（自动感知生命周期，简单方便）
viewModel.userName.observe(this) { name ->
    textView.text = name
}


// ============================================================
// 对比2：StateFlow 写法（推荐）
// ============================================================
class NewViewModel : ViewModel() {
    private val _userName = MutableStateFlow("")  // 必须给初始值
    val userName: StateFlow<String> = _userName.asStateFlow()

    fun loadUser() {
        viewModelScope.launch(Dispatchers.IO) {
            val user = repository.getUser()
            _userName.value = user.name   // 在协程中直接赋值（IO线程也可以）
        }
    }
}

// Activity 收集 StateFlow
lifecycleScope.launch {
    // ⚠️ 必须用 repeatOnLifecycle，否则退后台也会继续收集，浪费资源
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.userName.collect { name ->
            textView.text = name
        }
    }
}

// ❌ 错误写法（后台继续收集）
lifecycleScope.launch {
    viewModel.userName.collect { name ->   // 退后台了依然在执行，浪费资源
        textView.text = name
    }
}


// ============================================================
// StateFlow 进阶：combine 合并多个流
// ============================================================
class ProfileViewModel : ViewModel() {
    private val _userName = MutableStateFlow("")
    private val _userAge  = MutableStateFlow(0)

    // 把两个 StateFlow 合并成一个，任意一个变化时都触发
    val profileInfo: StateFlow<String> = combine(_userName, _userAge) { name, age ->
        "$name，$age 岁"
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = ""
    )
}
```

---

### Q7：Repository 层的职责是什么？

**答：**

Repository 是数据层的唯一入口，ViewModel 只和 Repository 交互，不关心数据从哪来（网络/数据库/缓存）。

```kotlin
// ============================================================
// 完整的数据层架构示例
// ============================================================

// --- 数据模型 ---
data class User(val id: Int, val name: String, val avatar: String)


// --- 远程数据源（网络）---
class UserRemoteDataSource(private val api: UserApi) {
    suspend fun fetchUser(id: Int): User = api.getUser(id)  // Retrofit 请求
}


// --- 本地数据源（数据库）---
class UserLocalDataSource(private val dao: UserDao) {
    suspend fun getUser(id: Int): User? = dao.findById(id)  // Room 查询
    suspend fun saveUser(user: User) = dao.insert(user)
    suspend fun updateUser(user: User) = dao.update(user)
}


// --- Repository：决定数据来源策略 ---
class UserRepository(
    private val remote: UserRemoteDataSource,
    private val local: UserLocalDataSource
) {
    /**
     * 策略：先查本地缓存，没有再请求网络，网络结果存入本地
     * ViewModel 完全不需要知道这些细节
     */
    suspend fun getUser(id: Int): User {
        // 第一步：查本地数据库
        val cached = local.getUser(id)
        if (cached != null) {
            return cached  // 有缓存直接返回，不请求网络（减少流量）
        }

        // 第二步：本地没有，请求网络
        val remote = remote.fetchUser(id)

        // 第三步：把网络结果存入本地（下次直接用缓存）
        local.saveUser(remote)

        return remote
    }

    /**
     * 策略：强制刷新，忽略缓存（下拉刷新时用）
     */
    suspend fun refreshUser(id: Int): User {
        val fresh = remote.fetchUser(id)
        local.updateUser(fresh)  // 更新本地缓存
        return fresh
    }

    /**
     * 返回 Flow：数据库有变化时自动推送新数据（Room 支持）
     */
    fun observeUser(id: Int): Flow<User?> = local.observeById(id)
    // 配合 StateFlow 使用，数据库更新 → UI 自动刷新，无需手动触发
}


// --- ViewModel：只调用 Repository，不关心数据从哪来 ---
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _user = MutableStateFlow<UiState<User>>(UiState.Loading)
    val user: StateFlow<UiState<User>> = _user.asStateFlow()

    fun loadUser(id: Int) {
        viewModelScope.launch {
            _user.value = UiState.Loading
            try {
                val user = repository.getUser(id)  // 只管调，不管实现
                _user.value = UiState.Success(user)
            } catch (e: Exception) {
                _user.value = UiState.Error(e.message ?: "加载失败")
            }
        }
    }
}
```

---

### Q8：如何封装 UI 状态（UiState）？

**答：**

用 `sealed class` 把所有 UI 状态统一封装，用 `when` 做分支处理，编译器会强制你处理所有情况。

```kotlin
// ============================================================
// UiState 封装 —— 三种状态覆盖所有场景
// ============================================================

// out T 表示协变，允许 UiState<User> 赋值给 UiState<Any>
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()                      // 加载中
    data class Success<T>(val data: T) : UiState<T>()       // 成功，携带数据
    data class Error(val message: String, val code: Int = -1) : UiState<Nothing>() // 失败，携带错误信息
}


// ============================================================
// ViewModel 完整示例
// ============================================================
class ArticleViewModel(private val repository: ArticleRepository) : ViewModel() {

    private val _articles = MutableStateFlow<UiState<List<Article>>>(UiState.Loading)
    val articles: StateFlow<UiState<List<Article>>> = _articles.asStateFlow()

    // 用于控制下拉刷新状态（独立出来，不混入主 UiState）
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()

    init {
        loadArticles()  // ViewModel 创建时自动加载
    }

    fun loadArticles() {
        viewModelScope.launch {
            _articles.value = UiState.Loading
            runCatching {
                // runCatching 是 Kotlin 的异常处理语法糖，等价于 try-catch
                withContext(Dispatchers.IO) { repository.getArticles() }
            }.onSuccess { list ->
                _articles.value = UiState.Success(list)
            }.onFailure { e ->
                _articles.value = UiState.Error(e.message ?: "加载失败")
            }
        }
    }

    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            runCatching {
                withContext(Dispatchers.IO) { repository.refreshArticles() }
            }.onSuccess { list ->
                _articles.value = UiState.Success(list)
            }.onFailure { e ->
                // 刷新失败时，保留原来的数据，只显示错误提示
                // 不要把整个页面变成 Error 状态
            }
            _isRefreshing.value = false
        }
    }
}


// ============================================================
// Fragment 中完整消费 UiState
// ============================================================
class ArticleFragment : Fragment() {
    private val viewModel: ArticleViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // 下拉刷新
        swipeRefreshLayout.setOnRefreshListener { viewModel.refresh() }

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {

                // 同时收集两个 Flow（并行）
                launch {
                    viewModel.articles.collect { state ->
                        when (state) {
                            is UiState.Loading       -> {
                                progressBar.visibility = View.VISIBLE
                                recyclerView.visibility = View.GONE
                                errorView.visibility = View.GONE
                            }
                            is UiState.Success       -> {
                                progressBar.visibility = View.GONE
                                recyclerView.visibility = View.VISIBLE
                                errorView.visibility = View.GONE
                                adapter.submitList(state.data)  // 更新列表
                            }
                            is UiState.Error         -> {
                                progressBar.visibility = View.GONE
                                recyclerView.visibility = View.GONE
                                errorView.visibility = View.VISIBLE
                                errorTextView.text = state.message
                            }
                        }
                    }
                }

                launch {
                    viewModel.isRefreshing.collect { refreshing ->
                        swipeRefreshLayout.isRefreshing = refreshing
                    }
                }
            }
        }
    }
}
```

---

## 模块三：Android 性能优化

---

### Q9：ANR 产生原因 + 排查步骤

**答：**

**ANR 触发条件（超时时间）：**

| 场景 | 超时 |
|------|------|
| 主线程未响应输入事件（触摸、按键）| 5 秒 |
| BroadcastReceiver.onReceive() 前台广播 | 10 秒 |
| BroadcastReceiver.onReceive() 后台广播 | 60 秒 |
| Service 前台启动 | 20 秒 |
| Service 后台启动 | 200 秒 |

```kotlin
// ============================================================
// ANR 常见原因 + 修复方案
// ============================================================

// ❌ 原因1：主线程做网络请求（最常见）
class BadActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ❌ 直接在主线程执行网络请求，必然 ANR
        val data = URL("https://api.example.com/data").readText()
        textView.text = data
    }
}

// ✅ 修复：放到协程 IO 线程
class GoodActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            val data = withContext(Dispatchers.IO) {
                URL("https://api.example.com/data").readText()  // IO 线程执行
            }
            textView.text = data  // 自动回主线程
        }
    }
}


// ❌ 原因2：主线程做数据库操作
class BadViewModel : ViewModel() {
    fun loadData() {
        val data = database.userDao().getAll()  // ❌ Room 默认禁止主线程操作，会直接崩溃
    }
}

// ✅ 修复：Room + 协程（Room 天然支持 suspend 函数）
class GoodViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = withContext(Dispatchers.IO) {
                database.userDao().getAll()  // ✅ IO 线程执行
            }
            _uiState.value = UiState.Success(data)
        }
    }
}


// ❌ 原因3：主线程等锁（死锁）
val lock = Object()
// 线程 A 持有 lockA，等待 lockB
// 线程 B 持有 lockB，等待 lockA
// 主线程等待其中一个 → ANR


// ============================================================
// ANR 排查工具：StrictMode（开发阶段使用，上线前移除）
// ============================================================
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {  // 只在 Debug 包开启
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()     // 检测主线程读磁盘
                    .detectDiskWrites()    // 检测主线程写磁盘
                    .detectNetwork()       // 检测主线程网络操作
                    .detectCustomSlowCalls() // 检测自定义耗时操作
                    .penaltyLog()          // 违规时打印日志
                    // .penaltyDeath()     // 违规时直接崩溃（更严格）
                    .build()
            )
        }
    }
}


// ============================================================
// ANR 发生后的排查步骤
// ============================================================

// 步骤1：复现 ANR，从设备拿取 traces 文件
// adb pull /data/anr/traces.txt ./traces.txt

// 步骤2：打开 traces.txt，找主线程调用栈
// 关键信息：
// "main" prio=5 tid=1 Blocked   ← 主线程处于 Blocked 状态
// at com.example.MyClass.doSomething(MyClass.kt:42)  ← 具体代码位置
// waiting to lock <0x12345678>  ← 等待哪个锁

// 步骤3：根据调用栈定位到具体代码，把耗时操作移到后台线程
```

---

### Q10：OOM 定位与解决

**答：**

**常见 OOM 原因：**
1. 大 Bitmap 未压缩直接加载
2. 内存泄漏（Activity 被长生命周期对象持有，无法回收）
3. 集合无限增长（缓存没有大小限制）
4. 频繁创建对象导致 GC 压力大

```kotlin
// ============================================================
// 原因1：Bitmap OOM + 修复方案
// ============================================================

// ❌ 错误：直接解码原图（一张 4000x3000 的照片约 46MB！）
fun loadImageBad(path: String): Bitmap {
    return BitmapFactory.decodeFile(path)  // 极可能 OOM
}

// ✅ 正确：按目标尺寸采样加载
fun loadImageGood(path: String, targetWidth: Int, targetHeight: Int): Bitmap {
    // 第一步：只读取图片尺寸，不实际加载像素（inJustDecodeBounds = true）
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true
    }
    BitmapFactory.decodeFile(path, options)
    // 此时 options.outWidth 和 options.outHeight 已有值，但没有加载像素

    // 第二步：计算采样率（inSampleSize）
    // 比如原图 4000x3000，目标 400x300，采样率 = 10
    // 采样后加载的像素是原来的 1/100，内存也是 1/100
    options.inSampleSize = calculateInSampleSize(options, targetWidth, targetHeight)

    // 第三步：按采样率实际加载
    options.inJustDecodeBounds = false
    return BitmapFactory.decodeFile(path, options)
}

fun calculateInSampleSize(options: BitmapFactory.Options, reqW: Int, reqH: Int): Int {
    val srcHeight = options.outHeight
    val srcWidth = options.outWidth
    var sampleSize = 1  // 默认采样率为 1（不缩小）

    if (srcHeight > reqH || srcWidth > reqW) {
        val halfH = srcHeight / 2
        val halfW = srcWidth / 2
        // 不断把采样率翻倍（1→2→4→8...），直到缩小后的尺寸 ≤ 目标尺寸
        while ((halfH / sampleSize) >= reqH && (halfW / sampleSize) >= reqW) {
            sampleSize *= 2
        }
    }
    return sampleSize  // 采样率必须是 2 的幂
}

// 实际项目直接用 Glide，它内部已做好采样和缓存
Glide.with(context)
    .load(url)
    .override(400, 300)   // 限制加载尺寸
    .into(imageView)


// ============================================================
// 原因2：内存泄漏 —— 常见场景和修复
// ============================================================

// 泄漏场景1：单例/静态变量持有 Activity Context
object ImageLoader {
    // ❌ 静态持有 Activity，导致 Activity 无法被 GC 回收
    var context: Context? = null

    // ✅ 应该持有 ApplicationContext（Application 的生命周期和 App 一样长，不会泄漏）
    lateinit var appContext: Context
    fun init(ctx: Context) {
        appContext = ctx.applicationContext  // 关键：取 applicationContext
    }
}

// 泄漏场景2：非静态内部类 Handler（持有外部 Activity 的引用）
class BadActivity : AppCompatActivity() {
    // ❌ 匿名内部类隐式持有 BadActivity.this 的引用
    // 如果 Handler 有延迟消息，Activity 销毁后，Handler 还活着，Activity 无法 GC
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            textView.text = "done"  // 直接访问外部 Activity 的成员
        }
    }
}

// ✅ 修复方案1：使用弱引用
class GoodActivity : AppCompatActivity() {
    private val handler = MyHandler(this)

    // 静态内部类 + 弱引用，不持有 Activity 强引用
    private class MyHandler(activity: GoodActivity) : Handler(Looper.getMainLooper()) {
        // WeakReference 不阻止 GC 回收 Activity
        private val activityRef = WeakReference(activity)

        override fun handleMessage(msg: Message) {
            val activity = activityRef.get() ?: return  // Activity 已销毁则直接返回
            activity.textView.text = "done"
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null)  // 移除所有消息
    }
}

// ✅ 修复方案2：现代写法，直接用协程替代 Handler
lifecycleScope.launch {
    delay(3000)
    textView.text = "done"
    // lifecycleScope 感知生命周期，Activity 销毁时自动取消，不会泄漏
}


// 泄漏场景3：未注销的监听器/回调
class SensorActivity : AppCompatActivity() {
    private lateinit var sensorManager: SensorManager

    override fun onResume() {
        super.onResume()
        sensorManager.registerListener(this, sensor, SENSOR_DELAY_UI)
    }

    override fun onPause() {
        super.onPause()
        sensorManager.unregisterListener(this)  // ✅ 必须注销，否则 Activity 无法 GC
    }
}


// ============================================================
// OOM 排查工具
// ============================================================
// 工具1：LeakCanary（自动检测内存泄漏，强烈推荐）
// build.gradle:
// debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.x'
// 不需要任何代码，自动检测，泄漏时弹出通知

// 工具2：Android Studio Memory Profiler
// Run → Profile → Memory Tab
// 1. 点击 GC（垃圾回收）按钮
// 2. 如果内存没有明显下降，说明有泄漏
// 3. 点击 Heap Dump，分析哪些对象还活着
```

---

### Q11：冷启动优化

**答：**

**冷启动流程：**
```
用户点击图标
→ 系统创建进程（Zygote fork，耗时）
→ Application.attachBaseContext()
→ Application.onCreate()（初始化各种 SDK，最容易在这里变慢）
→ MainActivity.onCreate()
→ 布局 inflate + 数据加载
→ 第一帧绘制完成 ← 用户看到内容（TTID）
```

```kotlin
// ============================================================
// 优化1：Application 中异步初始化非关键 SDK
// ============================================================
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        // 必须同步初始化（后续功能依赖它）
        initCrashSDK()       // 崩溃监控（上来就要）
        initNetworkSDK()     // 网络库（马上会用）

        // ✅ 异步初始化（不影响首屏展示）
        // MainScope 或 ProcessLifecycleOwner.get().lifecycleScope
        MainScope().launch(Dispatchers.IO) {
            initAnalyticsSDK()   // 统计分析（晚点初始化无所谓）
            initPushSDK()        // 推送（晚点没关系）
            initMapSDK()         // 地图（首页没有地图时可以延迟）
        }
    }
}


// ============================================================
// 优化2：SplashScreen API（Android 12+）消除白屏
// ============================================================
// res/values/themes.xml
// <style name="Theme.App.Starting" parent="Theme.SplashScreen">
//     <item name="windowSplashScreenBackground">@color/primary</item>
//     <item name="windowSplashScreenAnimatedIcon">@drawable/ic_logo</item>
//     <item name="postSplashScreenTheme">@style/Theme.App</item>
// </style>

// AndroidManifest.xml
// <activity android:theme="@style/Theme.App.Starting">

// MainActivity.kt
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // installSplashScreen 必须在 super.onCreate 之前调用
        val splashScreen = installSplashScreen()

        // 可以让 Splash 等待数据加载完毕再消失
        splashScreen.setKeepOnScreenCondition {
            !viewModel.isDataReady.value  // 数据没准备好就保持 Splash
        }

        super.onCreate(savedInstanceState)
    }
}


// ============================================================
// 优化3：SharedPreferences → DataStore（异步读写）
// ============================================================
// ❌ SharedPreferences 在主线程同步读写，影响启动速度
val sp = getSharedPreferences("config", MODE_PRIVATE)
val token = sp.getString("token", "")  // 主线程同步读，可能卡

// ✅ DataStore 异步读写（基于 Flow）
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "config")

val TOKEN_KEY = stringPreferencesKey("token")

// 读取（在协程中）
lifecycleScope.launch {
    val token = dataStore.data.map { it[TOKEN_KEY] ?: "" }.first()
}

// 写入（在协程中）
lifecycleScope.launch {
    dataStore.edit { settings ->
        settings[TOKEN_KEY] = newToken
    }
}


// ============================================================
// 优化4：RecyclerView 预加载（首页列表）
// ============================================================
// 在 ViewModel 的 init 中提前加载数据，不要等 UI 创建后再触发
class HomeViewModel : ViewModel() {
    init {
        loadArticles()  // ViewModel 创建时就开始加载，不等 Activity/Fragment 触发
    }
}


// ============================================================
// 测量启动时间
// ============================================================
// 方法1：ADB 命令
// adb shell am start -W com.example.app/.MainActivity
// 输出：TotalTime: 1250ms ← 这是冷启动时间

// 方法2：Logcat 过滤 "Displayed"
// ActivityManager: Displayed com.example.app/.MainActivity: +1s250ms

// 方法3：Perfetto（精细分析每个阶段耗时）
```

---

### Q12：RecyclerView 卡顿优化

**答：**

```kotlin
// ============================================================
// 优化1：使用 ListAdapter + DiffUtil（避免全量刷新）
// ============================================================

// ❌ 错误做法：全量刷新（每次都重新绘制所有 item）
adapter.notifyDataSetChanged()  // 所有 item 闪烁重绘，性能差

// ✅ 正确做法：ListAdapter 自动用 DiffUtil 计算差异，只更新变化的 item
data class Article(val id: Int, val title: String, val content: String)

class ArticleAdapter : ListAdapter<Article, ArticleAdapter.VH>(DIFF_CALLBACK) {

    companion object {
        // DiffUtil：定义如何比较两个 item
        val DIFF_CALLBACK = object : DiffUtil.ItemCallback<Article>() {
            // areItemsTheSame：是否是同一个数据项（用唯一 ID 比较）
            override fun areItemsTheSame(old: Article, new: Article) = old.id == new.id

            // areContentsTheSame：内容是否相同（ID 相同时进一步比较内容）
            // data class 自动生成 equals，直接用 == 即可
            override fun areContentsTheSame(old: Article, new: Article) = old == new
        }
    }

    class VH(view: View) : RecyclerView.ViewHolder(view) {
        val titleView: TextView = view.findViewById(R.id.title)
        val contentView: TextView = view.findViewById(R.id.content)

        fun bind(article: Article) {
            titleView.text = article.title
            contentView.text = article.content
            // ✅ bind 函数只做简单赋值，不做耗时操作
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_article, parent, false)
        return VH(view)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        // getItem(position) 是 ListAdapter 提供的方法，不要用自己维护的 list
        holder.bind(getItem(position))
    }
}

// 更新数据：submitList 内部异步计算 diff，不阻塞主线程
adapter.submitList(newList)


// ============================================================
// 优化2：onBindViewHolder 不做耗时操作
// ============================================================

// ❌ 错误：在 onBindViewHolder 中做复杂计算
override fun onBindViewHolder(holder: VH, position: Int) {
    val item = list[position]
    val formattedDate = SimpleDateFormat("yyyy-MM-dd").format(item.timestamp)  // ❌ 每次都创建对象
    val html = Html.fromHtml(item.content, Html.FROM_HTML_MODE_LEGACY)         // ❌ 耗时解析
    holder.titleView.text = html
    holder.dateView.text = formattedDate
}

// ✅ 正确：数据预处理放到 ViewModel 或 Repository
data class ArticleUiModel(
    val id: Int,
    val title: String,
    val formattedDate: String,  // ← 已格式化好
    val htmlContent: Spanned    // ← 已解析好
)

override fun onBindViewHolder(holder: VH, position: Int) {
    val item = getItem(position)
    holder.titleView.text = item.title          // ✅ 直接赋值，不计算
    holder.dateView.text = item.formattedDate   // ✅ 直接赋值，不计算
}


// ============================================================
// 优化3：图片加载用 Glide
// ============================================================
override fun onBindViewHolder(holder: VH, position: Int) {
    val item = getItem(position)
    Glide.with(holder.itemView.context)
        .load(item.imageUrl)
        .placeholder(R.drawable.ic_placeholder)  // 加载中显示占位图
        .error(R.drawable.ic_error)              // 加载失败显示错误图
        .override(200, 200)                      // 限制图片尺寸，节省内存
        .centerCrop()
        .into(holder.imageView)
}


// ============================================================
// 优化4：其他配置
// ============================================================
recyclerView.apply {
    setHasFixedSize(true)          // item 高度固定时，避免重复测量整个 RecyclerView

    // RecycledViewPool：多个 RecyclerView 共享 ViewHolder 缓存（如 ViewPager 内多页 RecyclerView）
    val pool = RecyclerView.RecycledViewPool()
    setRecycledViewPool(pool)

    // 预取（默认已开启，LinearLayoutManager 会提前准备下一个 item）
    // 不需要额外配置
}

// ============================================================
// 优化5：避免过度绘制（GPU 渲染）
// ============================================================
// 打开开发者选项 → 调试 GPU 过度绘制，看红色区域
// 红色 = 过度绘制 4 次以上，性能浪费
// 解决：去掉不必要的 background，减少 View 层叠
```

---

## 模块四：View 绘制 + 事件分发

---

### Q13：View 绘制三大流程

**答：**

**完整流程：** `measure（测量大小）→ layout（确定位置）→ draw（绘制内容）`

```kotlin
// ============================================================
// 完整自定义 View 示例：一个带文字的圆形进度条
// ============================================================
class CircleProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {

    // Paint 对象在 onDraw 外创建，避免每次绘制都 new 对象（GC 压力）
    private val bgPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 20f
        color = Color.LTGRAY
    }

    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 20f
        strokeCap = Paint.Cap.ROUND  // 圆角端点
        color = Color.BLUE
    }

    private val textPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        textSize = 60f
        color = Color.BLACK
        textAlign = Paint.Align.CENTER
    }

    var progress: Float = 0f   // 0.0 ~ 1.0
        set(value) {
            field = value.coerceIn(0f, 1f)  // 限制范围
            invalidate()  // 只是外观变化（颜色/内容），调用 invalidate 触发重绘
                          // 不需要调用 requestLayout（尺寸/位置没变）
        }

    // ============================================================
    // onMeasure：告诉父 View "我需要多大"
    // ============================================================
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        // MeasureSpec 包含 Mode 和 Size 两个信息
        val wMode = MeasureSpec.getMode(widthMeasureSpec)   // 模式
        val wSize = MeasureSpec.getSize(widthMeasureSpec)   // 父 View 给的最大值

        val desiredSize = 300  // 我期望的尺寸是 300px

        val width = when (wMode) {
            MeasureSpec.EXACTLY  -> wSize        // 父 View 指定了精确值（100dp 或 match_parent），就用它
            MeasureSpec.AT_MOST  -> minOf(desiredSize, wSize)  // wrap_content，不超过父 View 给的最大值
            else                 -> desiredSize  // UNSPECIFIED（ScrollView 内部），用期望值
        }

        // 同样处理高度（这里简化为宽高相等）
        setMeasuredDimension(width, width)  // 必须调用这个方法设置最终尺寸
    }

    // ============================================================
    // onLayout：确定自己在父 View 中的位置（自定义 View 通常不需要重写）
    //           自定义 ViewGroup 才需要在 onLayout 中摆放子 View
    // ============================================================

    // ============================================================
    // onDraw：实际绘制内容
    // ============================================================
    override fun onDraw(canvas: Canvas) {
        val cx = width / 2f       // 圆心 X
        val cy = height / 2f      // 圆心 Y
        val radius = minOf(cx, cy) - 30f  // 半径（留出边距）

        val oval = RectF(cx - radius, cy - radius, cx + radius, cy + radius)

        // 第1步：画灰色背景圆弧（完整 360 度）
        canvas.drawArc(oval, 0f, 360f, false, bgPaint)

        // 第2步：画蓝色进度圆弧（从 -90 度开始，即顶部，顺时针扫过 progress*360 度）
        canvas.drawArc(oval, -90f, progress * 360f, false, progressPaint)

        // 第3步：画进度文字
        val text = "${(progress * 100).toInt()}%"
        // drawText 的 y 坐标是基线位置，需要用 fontMetrics 计算居中
        val textY = cy - (textPaint.fontMetrics.ascent + textPaint.fontMetrics.descent) / 2
        canvas.drawText(text, cx, textY, textPaint)
    }
}

// 使用
circleProgressView.progress = 0.75f  // 显示 75%


// ============================================================
// requestLayout() vs invalidate() 区别
// ============================================================

// invalidate()：只触发 draw，用于外观变化（颜色、文字内容）
fun changeColor(newColor: Int) {
    paint.color = newColor
    invalidate()  // ← 只重绘，不重新测量和布局（高效）
}

// requestLayout()：触发 measure + layout + draw，用于尺寸/位置变化
fun changeSize(newSize: Int) {
    desiredSize = newSize
    requestLayout()  // ← 重新测量、布局、绘制
}

// postInvalidate()：在非主线程触发重绘（线程安全版本的 invalidate）
Thread {
    // 子线程更新数据后，需要触发 UI 重绘
    postInvalidate()  // ← 内部通过 Handler 切到主线程执行 invalidate
}.start()
```

---

### Q14：事件分发机制

**答：**

**分发链路：** `Activity → PhoneWindow → DecorView → 根 ViewGroup → 子 ViewGroup → View`

**三个核心方法：**

| 方法 | 存在于 | 返回 true 的含义 |
|------|--------|-----------------|
| `dispatchTouchEvent()` | Activity / ViewGroup / View | 事件被自己处理，不再向下传 |
| `onInterceptTouchEvent()` | 只有 ViewGroup 有 | 拦截事件，不传给子 View |
| `onTouchEvent()` | ViewGroup / View | 事件被自己消费 |

```kotlin
// ============================================================
// 事件分发流程图（文字版）
// ============================================================
//
// 手指按下 → Activity.dispatchTouchEvent()
//               ↓ 调用
//          ViewGroup.dispatchTouchEvent()
//               ↓ 先问
//          ViewGroup.onInterceptTouchEvent()
//          ├── 返回 true：拦截！自己处理
//          │       ↓
//          │   ViewGroup.onTouchEvent()  → 消费了，结束
//          │
//          └── 返回 false（默认）：不拦截，传给子 View
//                  ↓
//              View.dispatchTouchEvent()
//                  ↓
//              View.onTouchEvent()
//              ├── 返回 true：消费，结束
//              └── 返回 false：没消费，回传给父 ViewGroup.onTouchEvent()


// ============================================================
// 实战1：解决 ScrollView 内嵌 RecyclerView 滑动冲突
// ============================================================

// 方案1：禁用 RecyclerView 的嵌套滑动
recyclerView.isNestedScrollingEnabled = false
// 原理：RecyclerView 默认开启嵌套滑动，会优先消费滑动事件，导致外层 ScrollView 无法滑动
// 禁用后，事件传给外层 ScrollView 处理

// 方案2：自定义拦截（更灵活，可以按方向区分）
class MyScrollView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : ScrollView(context, attrs) {

    private var startX = 0f
    private var startY = 0f

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        return when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                startX = ev.x
                startY = ev.y
                super.onInterceptTouchEvent(ev)
            }
            MotionEvent.ACTION_MOVE -> {
                val dx = Math.abs(ev.x - startX)
                val dy = Math.abs(ev.y - startY)
                // 垂直滑动距离 > 水平滑动距离，说明是上下滑，ScrollView 拦截
                dy > dx
            }
            else -> super.onInterceptTouchEvent(ev)
        }
    }
}


// ============================================================
// 实战2：自定义 ViewGroup，理解 onInterceptTouchEvent
// ============================================================
class SwipeDeleteLayout @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {

    private var downX = 0f
    private val slop = ViewConfiguration.get(context).scaledTouchSlop  // 最小滑动判定距离

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        when (ev.action) {
            MotionEvent.ACTION_DOWN -> {
                downX = ev.x
                // DOWN 事件不拦截，让子 View 收到（否则子 View 的点击效果没有）
                return false
            }
            MotionEvent.ACTION_MOVE -> {
                val dx = ev.x - downX
                // 水平滑动超过阈值，拦截！由自己处理左右滑删除逻辑
                if (Math.abs(dx) > slop) {
                    return true  // 拦截后，子 View 收不到后续事件
                }
            }
        }
        return super.onInterceptTouchEvent(ev)
    }

    override fun onTouchEvent(ev: MotionEvent): Boolean {
        // 处理左右滑动逻辑
        when (ev.action) {
            MotionEvent.ACTION_MOVE -> {
                val dx = ev.x - downX
                translationX = dx  // 跟手移动
            }
            MotionEvent.ACTION_UP -> {
                if (translationX < -200f) {
                    // 滑动超过阈值，执行删除动画
                    animateDelete()
                } else {
                    // 回弹
                    animate().translationX(0f).start()
                }
            }
        }
        return true  // 返回 true 表示消费了事件
    }
}


// ============================================================
// 常见面试追问：DOWN 事件和 MOVE 事件的区别
// ============================================================
// 一个完整的手势 = ACTION_DOWN + N 个 ACTION_MOVE + ACTION_UP
//
// 关键规则：
// 1. 如果 onTouchEvent 在 DOWN 事件返回 false，后续 MOVE 和 UP 不会再来
// 2. 如果 onInterceptTouchEvent 在 DOWN 时返回 true，子 View 会收到 ACTION_CANCEL
// 3. MOVE 时才判断是否拦截，是因为 DOWN 时还不知道用户想干什么
```

---

# 第二优先级：面试题详细答案

---

## 模块五：Jetpack Compose

---

### Q15：Composable 重组（Recomposition）是什么？

**答：**

重组是 Compose 检测到状态变化后，重新执行 Composable 函数更新 UI 的过程。Compose 会智能跳过没有变化的部分，只重组受影响的 Composable。

```kotlin
// ============================================================
// 基础：状态驱动 UI
// ============================================================
@Composable
fun Counter() {
    // remember：在重组之间保留状态（不用 remember 的话，每次重组都会重置为 0）
    // mutableStateOf：创建可观察的状态，值变化时触发重组
    var count by remember { mutableStateOf(0) }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text(
            text = "点击次数：$count",
            fontSize = 24.sp
        )
        Button(onClick = { count++ }) {   // count++ 触发状态变化 → 触发重组
            Text("点击我")
        }
    }
    // 重组时，只有 Text("点击次数：$count") 和 count 相关的部分重组
    // Column 结构不变，不会整体重组
}


// ============================================================
// 重要：重组的"智能跳过"机制
// ============================================================
@Composable
fun Parent() {
    var name by remember { mutableStateOf("张三") }

    // name 变化时，Parent 重组
    // 但 ExpensiveChild 如果参数没变，会被跳过，不重组（性能优化）
    ExpensiveChild(title = "固定标题")  // ← 参数没变，跳过重组
    DynamicChild(name = name)           // ← 参数变了，重组
}

@Composable
fun ExpensiveChild(title: String) {
    // 只要 title 没变，这个函数就不会重新执行
    Text(title)
}


// ============================================================
// 常见陷阱：Lambda 导致不必要的重组
// ============================================================

// ❌ 错误：每次 Parent 重组，都会创建新的 lambda 对象，导致 Child 也重组
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }
    Child(onClick = { count++ })  // ← 每次重组都是新 lambda，Child 判断"参数变了"，重组
}

// ✅ 正确：用 remember 缓存 lambda
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }
    val onClick = remember { { count++ } }  // 缓存 lambda，重组时不会创建新对象
    Child(onClick = onClick)
}
```

---

### Q16：remember vs rememberSaveable 区别？

**答：**

| | `remember` | `rememberSaveable` |
|--|------------|-------------------|
| 重组时保留 | ✅ | ✅ |
| 屏幕旋转后保留 | ❌（重置）| ✅ |
| 进程被杀后恢复 | ❌ | ✅（存 Bundle）|
| 数据类型要求 | 任意 | 需要可序列化，或自定义 Saver |

```kotlin
// ============================================================
// 场景对比：搜索框内容在旋转屏幕后是否保留
// ============================================================

// ❌ remember：旋转后搜索框内容丢失，用户体验差
@Composable
fun SearchScreen() {
    var query by remember { mutableStateOf("") }   // 旋转后重置为 ""

    TextField(
        value = query,
        onValueChange = { query = it },
        placeholder = { Text("搜索...") }
    )
}

// ✅ rememberSaveable：旋转后内容依然保留
@Composable
fun SearchScreen() {
    var query by rememberSaveable { mutableStateOf("") }  // 旋转后恢复

    TextField(
        value = query,
        onValueChange = { query = it },
        placeholder = { Text("搜索...") }
    )
}


// ============================================================
// 复杂对象：自定义 Saver（无法自动序列化时）
// ============================================================
data class FilterState(val category: String, val minPrice: Int, val maxPrice: Int)

// 自定义 Saver：告诉 Compose 如何保存和恢复这个对象
val FilterStateSaver = Saver<FilterState, Bundle>(
    save = { state ->
        // 把对象保存到 Bundle
        Bundle().apply {
            putString("category", state.category)
            putInt("minPrice", state.minPrice)
            putInt("maxPrice", state.maxPrice)
        }
    },
    restore = { bundle ->
        // 从 Bundle 恢复对象
        FilterState(
            category = bundle.getString("category", "全部"),
            minPrice = bundle.getInt("minPrice", 0),
            maxPrice = bundle.getInt("maxPrice", 9999)
        )
    }
)

@Composable
fun FilterScreen() {
    var filterState by rememberSaveable(stateSaver = FilterStateSaver) {
        mutableStateOf(FilterState("全部", 0, 9999))
    }
    // filterState 现在在旋转后也能保留
}
```

---

### Q17：LaunchedEffect、SideEffect、DisposableEffect 使用场景？

**答：**

| API | 执行时机 | 是否有清理 | 典型场景 |
|-----|----------|------------|----------|
| `LaunchedEffect(key)` | 首次组合 + key 变化 | ✅ 协程取消 | 加载数据、启动动画 |
| `SideEffect` | 每次重组成功后 | ❌ | 同步状态到非 Compose 系统 |
| `DisposableEffect(key)` | 首次组合 + key 变化 | ✅ onDispose | 注册/注销监听器 |

```kotlin
// ============================================================
// LaunchedEffect：在 Composable 中安全启动协程
// ============================================================
@Composable
fun UserDetailScreen(userId: String) {
    val viewModel: UserDetailViewModel = viewModel()

    // key = userId：userId 变化时，老协程取消，新协程启动
    // 如果 key = Unit，则只在首次组合时执行一次
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)   // 在协程里调用 suspend 函数
    }

    // 显示 UI...
}

// 实战：倒计时
@Composable
fun CountdownTimer(totalSeconds: Int) {
    var remaining by remember { mutableStateOf(totalSeconds) }

    LaunchedEffect(totalSeconds) {      // totalSeconds 变化时重置倒计时
        remaining = totalSeconds
        while (remaining > 0) {
            delay(1000)                 // 等待 1 秒（可被取消）
            remaining--
        }
    }

    Text(text = "剩余：${remaining}秒")
}


// ============================================================
// SideEffect：每次重组成功后执行，同步到非 Compose 系统
// ============================================================
@Composable
fun AnalyticsScreen(screenName: String, analytics: Analytics) {

    // SideEffect 不能启动协程，不能做耗时操作
    // 用途：把 Compose 的状态同步给外部系统
    SideEffect {
        // 每次重组成功后上报，确保 analytics 和当前 screenName 同步
        analytics.setCurrentScreen(screenName)
    }

    // UI...
}


// ============================================================
// DisposableEffect：需要清理的副作用（注册/注销监听器）
// ============================================================
@Composable
fun NetworkStatusBanner() {
    var isConnected by remember { mutableStateOf(true) }
    val context = LocalContext.current

    // 注册网络监听器，离开页面时自动注销
    DisposableEffect(Unit) {
        val networkCallback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                isConnected = true
            }
            override fun onLost(network: Network) {
                isConnected = false
            }
        }

        val cm = context.getSystemService(ConnectivityManager::class.java)
        cm.registerDefaultNetworkCallback(networkCallback)  // 注册

        // onDispose：Composable 从组合树移除时（页面销毁时）执行
        onDispose {
            cm.unregisterNetworkCallback(networkCallback)   // 注销，防止内存泄漏
        }
    }

    if (!isConnected) {
        Text("无网络连接", color = Color.Red, modifier = Modifier.fillMaxWidth())
    }
}
```

---

## 模块六：蓝牙开发

---

### Q18：经典蓝牙 vs BLE 低功耗蓝牙区别？

**答：**

| 对比项 | 经典蓝牙（BR/EDR）| BLE（低功耗蓝牙）|
|--------|-----------------|---------------|
| 传输速率 | 高（1-3 Mbps）| 低（125Kbps-2Mbps）|
| 功耗 | 高（持续连接）| 极低（可用纽扣电池运行数年）|
| 连接延迟 | 较高（配对过程复杂）| 低（连接快）|
| 典型设备 | 蓝牙耳机、音箱、键盘 | 手环、戒指、心率计、温度计 |
| Android API | `BluetoothSocket`（像 Socket 通信）| `BluetoothGatt`（读写 Characteristic）|
| 数据模型 | 流式传输 | GATT：Service → Characteristic → Value |

**GATT 模型理解：**
```
设备（如心率戒指）
├── Service（功能模块，用 UUID 标识）
│   ├── 心率 Service（UUID: 0x180D）
│   │   └── Characteristic（具体数据点）
│   │       ├── 心率值（UUID: 0x2A37）← 可读、可通知
│   │       └── 传感器位置（UUID: 0x2A38）← 只读
│   └── 电量 Service（UUID: 0x180F）
│       └── 电量值（UUID: 0x2A19）← 可读
```

---

### Q19：BLE 完整连接流程（结合你的项目经验）

**答：**

```kotlin
// ============================================================
// BLE 完整开发流程（权限 → 扫描 → 连接 → 读写数据 → 断开）
// ============================================================

class BleManager(private val context: Context) {

    private val bluetoothAdapter: BluetoothAdapter? =
        (context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager).adapter

    private var bluetoothGatt: BluetoothGatt? = null

    // ============================================================
    // Step 1：检查权限（Android 12+ 需要 BLUETOOTH_SCAN 和 BLUETOOTH_CONNECT）
    // ============================================================
    fun checkPermissions(): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            // Android 12+
            context.checkSelfPermission(Manifest.permission.BLUETOOTH_SCAN) == PackageManager.PERMISSION_GRANTED &&
            context.checkSelfPermission(Manifest.permission.BLUETOOTH_CONNECT) == PackageManager.PERMISSION_GRANTED
        } else {
            // Android 11 及以下
            context.checkSelfPermission(Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED
        }
    }

    // ============================================================
    // Step 2：扫描 BLE 设备（按服务 UUID 过滤，精准找目标设备）
    // ============================================================
    private val RING_SERVICE_UUID = UUID.fromString("0000180D-0000-1000-8000-00805f9b34fb")

    fun startScan(onDeviceFound: (BluetoothDevice) -> Unit) {
        val scanner = bluetoothAdapter?.bluetoothLeScanner ?: return

        // 按 UUID 过滤，只扫描有心率服务的设备（避免扫到所有 BLE 设备）
        val filters = listOf(
            ScanFilter.Builder()
                .setServiceUuid(ParcelUuid(RING_SERVICE_UUID))
                .build()
        )

        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)  // 低延迟模式，扫描快
            .build()

        val scanCallback = object : ScanCallback() {
            override fun onScanResult(callbackType: Int, result: ScanResult) {
                val device = result.device
                // 找到目标设备，停止扫描并回调
                scanner.stopScan(this)
                onDeviceFound(device)
            }

            override fun onScanFailed(errorCode: Int) {
                // 扫描失败处理（如蓝牙未开启：ERROR_BLUETOOTH_DISABLED）
            }
        }

        scanner.startScan(filters, settings, scanCallback)
    }

    // ============================================================
    // Step 3：连接设备 + 处理 GATT 回调
    // ============================================================
    private val MY_CHAR_UUID = UUID.fromString("00002A37-0000-1000-8000-00805f9b34fb")  // 心率值

    fun connect(device: BluetoothDevice, onDataReceived: (ByteArray) -> Unit) {
        bluetoothGatt = device.connectGatt(
            context,
            false,           // autoConnect=false：直接连接，不等设备进入范围
            object : BluetoothGattCallback() {

                // 连接状态变化回调
                override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
                    when (newState) {
                        BluetoothProfile.STATE_CONNECTED -> {
                            // 连接成功！立刻发现服务
                            gatt.discoverServices()
                        }
                        BluetoothProfile.STATE_DISCONNECTED -> {
                            // 断开连接，清理资源
                            gatt.close()
                        }
                    }
                }

                // 服务发现完成回调（discoverServices() 的结果）
                override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
                    if (status != BluetoothGatt.GATT_SUCCESS) return

                    // 找到心率服务下的心率值 Characteristic
                    val service = gatt.getService(UUID.fromString("0000180D-0000-1000-8000-00805f9b34fb"))
                    val characteristic = service?.getCharacteristic(MY_CHAR_UUID) ?: return

                    // 开启通知：设备有新数据时主动推过来（不需要轮询）
                    gatt.setCharacteristicNotification(characteristic, true)

                    // 还需要写 CCCD（Client Characteristic Configuration Descriptor）才能真正开启通知
                    val descriptor = characteristic.getDescriptor(
                        UUID.fromString("00002902-0000-1000-8000-00805f9b34fb")  // CCCD 的固定 UUID
                    )
                    descriptor?.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
                    gatt.writeDescriptor(descriptor)
                }

                // 收到设备推送的数据（开启通知后，每次心率变化都会触发）
                override fun onCharacteristicChanged(
                    gatt: BluetoothGatt,
                    characteristic: BluetoothGattCharacteristic
                ) {
                    val rawData = characteristic.value  // 原始字节数组
                    onDataReceived(rawData)             // 回调给上层解析
                }

                // 主动读数据的回调（gatt.readCharacteristic() 的结果）
                override fun onCharacteristicRead(
                    gatt: BluetoothGatt,
                    characteristic: BluetoothGattCharacteristic,
                    status: Int
                ) {
                    if (status == BluetoothGatt.GATT_SUCCESS) {
                        val value = characteristic.value
                        // 处理读到的数据
                    }
                }
            },
            BluetoothDevice.TRANSPORT_LE  // 指定使用 BLE 传输（Android 6+）
        )
    }

    // ============================================================
    // Step 4：断开连接，释放资源
    // ============================================================
    fun disconnect() {
        bluetoothGatt?.disconnect()  // 先断开
        bluetoothGatt?.close()       // 再释放（close 之后 gatt 不可再用）
        bluetoothGatt = null
    }
}

// ============================================================
// 解析心率数据（BLE 心率 Characteristic 的数据格式是标准的）
// ============================================================
fun parseHeartRate(rawData: ByteArray): Int {
    // 第 0 个字节是 Flags，bit0=0 表示心率值是 1 字节，bit0=1 表示是 2 字节
    val flags = rawData[0].toInt()
    return if (flags and 0x01 == 0) {
        rawData[1].toInt() and 0xFF   // 心率值在第 1 字节（单位：bpm）
    } else {
        (rawData[2].toInt() and 0xFF shl 8) or (rawData[1].toInt() and 0xFF)  // 2 字节
    }
}
```

---

## 模块七：Web3 区块链

---

### Q20：WalletConnect 授权流程？

**答：**

```kotlin
// ============================================================
// WalletConnect 流程理解（配合文字说明）
// ============================================================

// 流程：
// 1. App 生成一个唯一的连接 URI（包含 topic 和配对密钥）
// 2. 用户用钱包 APP 扫描 URI（或点击 Deep Link）
// 3. 两端通过 WalletConnect 中继服务器建立加密 WebSocket 通道
// 4. App 发起 session_proposal（我想访问你的以太坊账号）
// 5. 用户在钱包 APP 点击"同意"
// 6. 钱包返回授权的账号地址列表
// 7. 后续 App 发起签名请求（如交易、消息签名），用户在钱包 APP 确认

// ============================================================
// 代码示例（WalletConnect v2）
// ============================================================

// 第一步：初始化
val coreClient = CoreClient
CoreClient.initialize(
    relayServerUrl = "wss://relay.walletconnect.com?projectId=YOUR_PROJECT_ID",
    connectionType = ConnectionType.AUTOMATIC,
    application = application,
    metaData = Core.Model.AppMetaData(
        name = "My DApp",
        description = "A sample dapp",
        url = "https://example.com",
        icons = listOf("https://example.com/icon.png"),
        redirect = "kotlin-wallet-connect-wc://request"  // 用于唤起钱包 APP
    )
)

Web3Wallet.initialize(Wallet.Params.Init(core = CoreClient))

// 第二步：监听来自 DApp 的连接请求
Web3Wallet.setWalletDelegate(object : Web3Wallet.WalletDelegate {
    override fun onSessionProposal(sessionProposal: Wallet.Model.SessionProposal, verifyContext: Wallet.Model.VerifyContext) {
        // 用户收到连接请求，展示确认对话框
        val dappName = sessionProposal.name         // DApp 名称
        val dappDescription = sessionProposal.description

        // 用户点击同意后，批准 session
        val sessionNamespaces = mapOf(
            "eip155" to Wallet.Model.Namespace.Session(
                accounts = listOf("eip155:1:0xYOUR_WALLET_ADDRESS"),  // 授权的账号
                methods = listOf("eth_sendTransaction", "personal_sign"),  // 授权的操作
                events = listOf("chainChanged", "accountsChanged")
            )
        )
        val approveParams = Wallet.Params.SessionApprove(
            proposerPublicKey = sessionProposal.proposerPublicKey,
            namespaces = sessionNamespaces
        )
        Web3Wallet.approveSession(approveParams) { error ->
            // 处理错误
        }
    }

    override fun onSessionRequest(sessionRequest: Wallet.Model.SessionRequest, verifyContext: Wallet.Model.VerifyContext) {
        // 收到 DApp 的操作请求（如签名、发交易）
        when (sessionRequest.request.method) {
            "personal_sign" -> {
                // 展示签名确认对话框，用户同意后签名并返回结果
                val message = sessionRequest.request.params
                val signature = signMessage(message)

                val response = Wallet.Params.SessionRequestResponse(
                    sessionTopic = sessionRequest.topic,
                    jsonRpcResponse = Wallet.Model.JsonRpcResponse.JsonRpcResult(
                        id = sessionRequest.request.id,
                        result = signature
                    )
                )
                Web3Wallet.respondSessionRequest(response) { error -> }
            }
        }
    }
})
```

---

### Q21：web3j 智能合约交互？

**答：**

```kotlin
// ============================================================
// web3j：Android 上与以太坊交互的 Java/Kotlin 库
// ============================================================

// 引入依赖
// implementation 'org.web3j:core:4.8.9-android'

// ============================================================
// Step 1：连接以太坊节点
// ============================================================
// 节点是以太坊网络的入口，可以用 Infura/Alchemy 等服务商提供的节点
val web3j = Web3j.build(HttpService("https://mainnet.infura.io/v3/YOUR_PROJECT_ID"))

// 测试连接是否成功
val blockNumber = web3j.ethBlockNumber().send().blockNumber
println("当前区块高度：$blockNumber")


// ============================================================
// Step 2：创建钱包账号（Credentials）
// ============================================================
// 方式1：从私钥创建（私钥不能泄露！）
val credentials = Credentials.create("0xYOUR_PRIVATE_KEY")
val address = credentials.address   // 钱包地址

// 方式2：从助记词创建
val mnemonic = "word1 word2 ... word12"
val seed = MnemonicUtils.generateSeed(mnemonic, "")
val masterKeypair = Bip32ECKeyPair.generateKeyPair(seed)
val childKeypair = Bip32ECKeyPair.deriveKeyPair(masterKeypair, intArrayOf(44 or HARDENED, 60 or HARDENED, 0 or HARDENED, 0, 0))
val credentials2 = Credentials.create(childKeypair)


// ============================================================
// Step 3：查询余额
// ============================================================
suspend fun getEthBalance(address: String): BigDecimal = withContext(Dispatchers.IO) {
    val balanceWei = web3j
        .ethGetBalance(address, DefaultBlockParameterName.LATEST)
        .send()
        .balance
    // Wei 是最小单位，1 ETH = 10^18 Wei
    Convert.fromWei(balanceWei.toBigDecimal(), Convert.Unit.ETHER)
}


// ============================================================
// Step 4：调用合约（读取，不需要 Gas，不需要签名）
// ============================================================
// 假设有一个 ERC-20 代币合约，查询余额
// 需要先用 web3j codegen 从 ABI 生成 Java 包装类，这里用手动调用演示

val contractAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"  // USDC 合约

// 构造 Function（合约函数名 + 参数类型 + 返回类型）
val function = Function(
    "balanceOf",                                     // 函数名
    listOf(Address(address)),                        // 参数：查询谁的余额
    listOf(object : TypeReference<Uint256>() {})     // 返回类型：uint256
)

// 编码调用数据
val encodedFunction = FunctionEncoder.encode(function)

// 发送 call（不上链，不消耗 gas）
val response = web3j.ethCall(
    Transaction.createEthCallTransaction(address, contractAddress, encodedFunction),
    DefaultBlockParameterName.LATEST
).send()

// 解码返回值
val decoded = FunctionReturnDecoder.decode(response.value, function.outputParameters)
val usdcBalance = (decoded[0] as Uint256).value
println("USDC 余额：${usdcBalance.divide(BigInteger.TEN.pow(6))} USDC")  // USDC 精度是 6


// ============================================================
// Step 5：发送交易（写入区块链，需要签名 + Gas）
// ============================================================
suspend fun transferETH(
    toAddress: String,
    amountEth: BigDecimal
) = withContext(Dispatchers.IO) {
    val nonce = web3j.ethGetTransactionCount(credentials.address, DefaultBlockParameterName.LATEST)
        .send().transactionCount

    val amountWei = Convert.toWei(amountEth, Convert.Unit.ETHER).toBigInteger()

    val gasPrice = web3j.ethGasPrice().send().gasPrice  // 当前 Gas 价格
    val gasLimit = BigInteger.valueOf(21000)             // 普通转账固定 21000 Gas

    val rawTransaction = RawTransaction.createEtherTransaction(
        nonce, gasPrice, gasLimit, toAddress, amountWei
    )

    // 用私钥签名交易
    val signedTransaction = TransactionEncoder.signMessage(rawTransaction, credentials)
    val hexValue = Numeric.toHexString(signedTransaction)

    // 广播到网络
    val txHash = web3j.ethSendRawTransaction(hexValue).send().transactionHash
    println("交易哈希：$txHash")
    // 可以在 etherscan.io 查询这个交易哈希确认状态
}
```

---

## 高频手撕代码题

---

### 手撕1：LRU 缓存（链接最近最少使用）

```kotlin
// ============================================================
// LRU 原理：最近使用的放最前面，超出容量时删除最久未使用的（最后面）
// 用 LinkedHashMap(accessOrder=true) 实现：get/put 后自动把该项移到末尾
// ============================================================
class LRUCache(private val capacity: Int) {

    // LinkedHashMap 第三个参数 accessOrder=true：
    // 每次 get 或 put 一个 key，该 key 自动移到链表末尾（最近使用）
    // 链表头部就是最久未使用的
    private val cache = object : LinkedHashMap<Int, Int>(capacity, 0.75f, true) {
        // 当 size > capacity 时，removeEldestEntry 返回 true，自动删除最老的（头部）
        override fun removeEldestEntry(eldest: MutableMap.MutableEntry<Int, Int>): Boolean {
            return size > capacity
        }
    }

    fun get(key: Int): Int {
        return cache.getOrDefault(key, -1)  // 没找到返回 -1
        // 注意：get 操作也会把这个 key 移到链表末尾（最近使用）
    }

    fun put(key: Int, value: Int) {
        cache[key] = value  // put 也会移到末尾，如果超容量自动删除头部
    }
}

// 使用示例：
// val lru = LRUCache(3)
// lru.put(1, 1)   // cache: {1=1}
// lru.put(2, 2)   // cache: {1=1, 2=2}
// lru.put(3, 3)   // cache: {1=1, 2=2, 3=3}
// lru.get(1)      // 访问1，cache: {2=2, 3=3, 1=1}  1 移到末尾
// lru.put(4, 4)   // 容量满，删除最久未使用的 2，cache: {3=3, 1=1, 4=4}
// lru.get(2)      // returns -1（2 已被删除）
```

---

### 手撕2：单例模式（双重检验锁）

```kotlin
// ============================================================
// 为什么需要双重检验锁？
// ============================================================
// 问题：多线程环境下，普通单例会创建多个实例
// 解法：用 synchronized 加锁，但每次获取都加锁性能差
// 优化：先判断（不加锁），null 时才加锁再判断（双重检验）

class NetworkClient private constructor() {

    fun request(url: String) { /* ... */ }

    companion object {
        // @Volatile：保证 instance 对所有线程可见（禁止 CPU 指令重排）
        // 没有 @Volatile：线程 A 写入 instance，线程 B 可能读到旧值（缓存）
        @Volatile
        private var instance: NetworkClient? = null

        fun getInstance(): NetworkClient {
            // 第一次检验：instance 不为 null 直接返回（不加锁，高性能）
            return instance ?: synchronized(this) {
                // 进入 synchronized 后，第二次检验：
                // 可能线程 A 和线程 B 同时通过了第一次检验
                // A 先拿到锁创建了实例，B 等待后拿到锁
                // 第二次检验确保 B 不会再创建一次
                instance ?: NetworkClient().also { instance = it }
            }
        }
    }
}

// Kotlin 更简洁的单例写法（推荐）
object NetworkClientKt {
    // object 关键字：Kotlin 内置单例，线程安全，懒加载
    // 等价于 Java 的静态内部类单例写法（最推荐）
    fun request(url: String) { /* ... */ }
}
```

---

### 手撕3：观察者模式（类比 LiveData/StateFlow）

```kotlin
// ============================================================
// 观察者模式：发布-订阅，解耦数据变化和 UI 更新
// ============================================================

// 观察者接口（订阅方需要实现）
interface Observer<T> {
    fun onChanged(data: T)
}

// 被观察者（数据持有者）
class Observable<T> {
    // 存储所有观察者
    private val observers = mutableListOf<Observer<T>>()

    // value 的 setter：值变化时通知所有观察者
    var value: T? = null
        set(newValue) {
            field = newValue
            if (newValue != null) {
                notifyObservers(newValue)  // 通知所有订阅者
            }
        }

    fun addObserver(observer: Observer<T>) {
        observers.add(observer)
    }

    fun removeObserver(observer: Observer<T>) {
        observers.remove(observer)
    }

    private fun notifyObservers(data: T) {
        observers.forEach { it.onChanged(data) }
    }
}

// 使用示例
class UserRepository {
    val currentUser = Observable<String>()   // 被观察者

    fun login(name: String) {
        currentUser.value = name  // 值变化，自动通知所有观察者
    }
}

class MainActivity : AppCompatActivity() {
    private val repository = UserRepository()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 订阅：当 currentUser 变化时，更新 UI
        repository.currentUser.addObserver(object : Observer<String> {
            override fun onChanged(data: String) {
                usernameTextView.text = data  // 自动更新
            }
        })

        // 触发数据变化
        repository.login("张三")  // → 自动通知 → usernameTextView 更新为 "张三"
    }

    override fun onDestroy() {
        super.onDestroy()
        // 记得注销观察者，防止内存泄漏（LiveData 自动处理这个问题）
    }
}
```

---

## 面试小贴士

1. **STAR 法则** 回答项目经历：Situation（背景）→ Task（任务）→ Action（行动）→ Result（结果）
2. **不会的题不要沉默**，说"我没做过，但我的思路是..."比沉默好
3. **时间线重叠问题** 提前准备回答：Tippot 是 Flutter 项目，社书是 Android 项目，两个同期并行开发
4. **AI 工具经验** 是差异化亮点，主动提，讲清楚 Claude Code + CLAUDE.md + MCP + Figma 的工作流
5. **薪资谈判** 先让对方出价，区间 25-40K，不要自己先说具体数字

---

*文件生成时间：2026-03-05 | 对应简历：何先生，8 年 Android 经验*
