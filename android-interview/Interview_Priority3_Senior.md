# Android 面试备考手册 · 第三优先级

> 面向高级工程师 / 技术负责人岗位，覆盖管理经验、技术决策、架构设计、算法
> 更新日期：2026-03-05

---

# 一、管理岗必考：行为面试题（STAR 法则）

> STAR = Situation（背景）→ Task（任务）→ Action（行动）→ Result（结果）
> 每个故事控制在 2-3 分钟内讲完，结果尽量量化。

---

## 故事1：技术选型决策 —— 为什么引入 Kotlin Coroutines 替换 RxJava

**面试官问法：**
- "你做过哪些技术选型？决策过程是怎样的？"
- "你如何在团队推行新技术？"

**STAR 回答模板：**

```
【Situation - 背景】
在 YLS 项目初期，团队网络层和异步处理都用 RxJava。
随着项目规模增大，出现了几个问题：
1. RxJava 学习曲线陡，新人上手慢，经常写出 subscribe 忘记 dispose 的内存泄漏
2. 操作符嵌套多了以后代码可读性下降
3. 错误处理链路复杂

【Task - 任务】
我作为技术负责人，需要决策是否切换技术栈，以及如何平稳过渡。

【Action - 行动】
1. 评估阶段（2周）
   - 写 POC（概念验证）代码，对比 RxJava 和 Kotlin Coroutines+Flow 在相同业务场景下的代码量和可读性
   - 调研团队成员对两种技术的熟悉程度
   - 查阅 Google 官方推荐（Android 官方 2022 年已全面推荐 Coroutines）
   - 评估迁移成本

2. 决策阶段
   - 新功能模块全部用 Coroutines+Flow
   - 老代码分批迁移，不强制一次性全改
   - RxJava 和 Coroutines 并存期间，写统一的桥接层（.asFlow()）

3. 推行阶段
   - 写内部技术文档和示例代码
   - 对团队做 1 次技术分享（约 1 小时）
   - Code Review 时重点关注协程使用是否规范
   - 建立 CLAUDE.md 文件，让 AI 辅助生成的代码也遵循协程规范

【Result - 结果】
- 新同学上手速度明显加快，协程写法比 RxJava 直观很多
- 内存泄漏问题（忘记 dispose）几乎消失，因为 lifecycleScope/viewModelScope 自动管理
- 代码行数减少约 30%（同等功能）
- 整个迁移过程持续约 3 个月，业务没有受到影响
```

---

## 故事2：AI 工具落地 —— Claude Code + MCP + Figma 提升研发效率

**面试官问法：**
- "你们团队如何使用 AI 工具提效的？"
- "你有没有做过工程效率方面的改进？"

**STAR 回答模板：**

```
【Situation - 背景】
团队 3 人 Android 工程师，需求量大，经常加班。
设计稿到代码的转换、ViewModel 业务逻辑编写是最耗时的两个环节。

【Task - 任务】
探索 AI 辅助编程落地方案，目标：减少重复性编码工作，让工程师聚焦核心逻辑。

【Action - 行动】

1. 建立 CLAUDE.md 项目规范文件
   在项目根目录写 CLAUDE.md，定义：
   - 架构约定（MVVM 分层规则，Repository 使用规范）
   - 代码风格（命名规范、注释规范）
   - 禁用项（禁止在 UI 层直接调用网络，禁止 GlobalScope）
   - 常用工具类路径（如统一的网络封装、Toast 工具等）

   效果：Claude Code 生成的代码自动对齐项目规范，不需要大量修改

2. MCP + Figma 设计稿转代码
   配置 Figma MCP 工具，让 Claude Code 能直接读取 Figma 设计稿
   工作流：
   设计师出稿 → 工程师在 Claude Code 里发送 Figma 链接 →
   Claude Code 读取设计规范（颜色、字体、间距）→ 生成对应的 XML 或 Compose 代码

3. AI 辅助生成 ViewModel 业务逻辑
   把接口文档粘贴给 Claude Code，描述业务需求
   Claude Code 生成：ViewModel、UiState、Repository 调用骨架
   工程师主要做：业务逻辑验证、边界情况处理

4. AI 辅助生成单元测试
   Claude Code 根据 ViewModel 代码自动生成对应的单元测试
   覆盖 Success/Error/Loading 三种状态的测试用例

【Result - 结果】
- 设计稿到基础 UI 代码的时间从平均 2-3 小时降至 30-40 分钟
- ViewModel 骨架代码生成速度提升约 50%
- 单元测试覆盖率从 20% 提升到 60%+（因为生成成本低了，工程师愿意写了）
- 团队整体需求交付速度提升约 30%
```

---

## 故事3：遇到的最大技术挑战

**面试官问法：**
- "讲一个你解决过的最难的技术问题"
- "项目中遇到过什么印象深刻的 Bug？"

**STAR 回答模板：**

```
【Situation - 背景】
健康 APP 上线后，部分用户反馈数据同步有延迟，有时心率数据更新不及时。
而且这个问题难以复现，只在特定机型（华为、小米）上出现。

【Task - 任务】
定位并修复 BLE 蓝牙数据同步的稳定性问题。

【Action - 行动】

1. 收集信息
   - 在 Firebase Crashlytics 查看是否有相关崩溃（没有崩溃，是数据延迟）
   - 给用户加埋点，记录 BLE 连接状态、数据接收时间戳、onCharacteristicChanged 回调次数
   - 找到有问题的用户日志分析

2. 定位问题
   分析日志发现两个问题：
   a. 华为/小米对 BLE 连接有省电策略，后台会主动断开连接，但回调不及时
   b. onCharacteristicChanged 回调有时在主线程，有时在子线程（设备固件差异），
      导致偶发的线程安全问题

3. 修复方案
   a. 添加 BLE 心跳机制：每 30 秒发一次 read 请求，保持连接活跃
   b. 注册 BluetoothGattCallback.onConnectionStateChange，断连后自动重连（最多 3 次）
   c. 所有 Characteristic 数据处理统一切换到 Dispatchers.IO，避免主线程操作
   d. 用 StateFlow 管理连接状态，UI 层实时显示连接状态

4. 验证
   在 10 台不同品牌测试机上跑 72 小时压测
   用 ADB 日志监控连接断开次数和数据接收频率

【Result - 结果】
- 数据延迟问题基本消失（从偶发延迟 5-10 秒到几乎无感知）
- 用户投诉量下降 90%
- 顺带发现了线程安全问题，修复后 App 稳定性提升
- 总结成内部文档：《BLE 开发稳定性最佳实践》
```

---

## 故事4：Code Review 和带人经验

**面试官问法：**
- "你如何做 Code Review？"
- "你是怎么带新人的？"

**STAR 回答模板：**

```
【带人方式】
1. 新人入职：提供规范文档（架构说明 + CLAUDE.md）+ 1 对 1 讲解核心模块
2. 前 2 周：小需求入手，我做 Reviewer，重点关注架构规范和代码习惯
3. Code Review 原则：
   - 重点看：线程安全、内存泄漏、架构分层是否正确
   - 不苛求：命名风格、注释等次要问题
   - 方式：不直接改代码，而是在评论里解释"为什么"，让对方自己改
   - 好的代码也要表扬，不只挑问题

4. 常见 Review 问题（举例）：

   ❌ 问题1：在 Fragment 中直接 lifecycleScope.launch { viewModel.flow.collect{} }
   ✅ 应改为：repeatOnLifecycle(STARTED)，否则后台继续收集

   ❌ 问题2：在 Repository 里 new Handler(Looper.getMainLooper()) 切主线程
   ✅ 应改为：让调用方在协程里 withContext(Main) 切换

   ❌ 问题3：onBindViewHolder 里调用 Html.fromHtml() 解析 HTML
   ✅ 应在数据预处理阶段（ViewModel/Repository）处理好，bind 时直接赋值

【结果】
新人 3 个月后基本能独立开发标准模块，半年后能独立负责一个完整业务线
```

---

# 二、架构设计：模块化 / 组件化

---

## Q：什么是模块化？为什么要做模块化？

**答：**

模块化是把一个大 APP 拆分成多个相对独立的 Gradle Module，每个模块负责一个功能域。

**为什么做：**
- 编译速度：只修改了一个模块，只重新编译这个模块（Gradle 增量编译）
- 团队协作：不同模块不同人负责，减少代码冲突
- 代码复用：基础模块（网络、UI 组件）可以被多个业务模块复用
- 可测试性：模块边界清晰，单独测试更容易

```
// ============================================================
// 典型模块化结构（以健康 APP 为例）
// ============================================================

MyHealthApp/
├── app/                     # 壳模块，只负责组装各业务模块（Application + 路由初始化）
│
├── feature/                 # 业务功能模块
│   ├── health/              # 健康数据模块（步数、心率、睡眠）
│   ├── ai_assistant/        # AI 健康助手模块
│   ├── profile/             # 个人中心模块
│   └── social/              # 社交功能模块
│
├── core/                    # 核心基础模块
│   ├── network/             # 网络层（OkHttp + Retrofit 封装）
│   ├── database/            # 数据库（Room）
│   ├── ui_components/       # 通用 UI 组件（自定义 View、主题）
│   ├── bluetooth/           # 蓝牙封装
│   └── common/              # 通用工具类、扩展函数
│
└── domain/                  # 领域层（可选，业务实体 + 业务逻辑接口）
    ├── model/               # 业务模型（User、HealthRecord 等）
    └── repository/          # Repository 接口定义


// ============================================================
// 模块间依赖规则（很重要！）
// ============================================================

// 允许的依赖方向：
// app → feature → core
// feature 之间不能直接依赖（通过路由跳转，如 ARouter）

// build.gradle 示例
// feature/health/build.gradle
dependencies {
    implementation(project(":core:network"))      // ✅ 依赖核心网络层
    implementation(project(":core:database"))     // ✅ 依赖核心数据库
    implementation(project(":domain"))            // ✅ 依赖领域模型

    // implementation(project(":feature:profile"))  // ❌ 不允许 feature 间直接依赖
}


// ============================================================
// 模块间通信：ARouter（阿里路由框架）
// ============================================================

// 在 feature/health 模块中跳转到 feature/profile
@Route(path = "/profile/main")   // profile 模块暴露的路由
class ProfileActivity : AppCompatActivity()

// health 模块中跳转（不需要 import ProfileActivity）
ARouter.getInstance()
    .build("/profile/main")
    .withString("userId", "123")   // 传递参数
    .navigation()

// 这样 health 模块完全不知道 ProfileActivity 的存在，解耦彻底
```

---

## Q：如何设计网络层封装？

**答：**

```kotlin
// ============================================================
// 完整的网络层封装（core/network 模块）
// ============================================================

// ---- 1. API 接口定义（Retrofit）----
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): Response<UserDto>

    @POST("users/login")
    suspend fun login(@Body request: LoginRequest): Response<LoginResponse>
}


// ---- 2. 统一响应体（服务端数据结构）----
data class ApiResponse<T>(
    val code: Int,        // 业务状态码（200=成功，401=未登录，etc.）
    val message: String,
    val data: T?
)


// ---- 3. 网络结果封装 ----
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>()
    object NetworkException : NetworkResult<Nothing>()  // 网络不通
}


// ---- 4. OkHttp 拦截器 ----
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()

        // 自动给每个请求加 Authorization Header
        val newRequest = originalRequest.newBuilder()
            .header("Authorization", "Bearer ${tokenProvider.getToken()}")
            .header("Accept-Language", Locale.getDefault().language)
            .build()

        val response = chain.proceed(newRequest)

        // 处理 401（Token 过期）
        if (response.code == 401) {
            // 触发 Token 刷新逻辑（这里简化）
            tokenProvider.clearToken()
            // 实际可以加 Token 刷新后重试机制
        }

        return response
    }
}

class LoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.currentTimeMillis()

        val response = chain.proceed(request)

        val duration = System.currentTimeMillis() - startTime
        // Debug 环境打印请求日志
        if (BuildConfig.DEBUG) {
            println("${request.method} ${request.url} → ${response.code} (${duration}ms)")
        }

        return response
    }
}


// ---- 5. Retrofit 和 OkHttp 组装 ----
object NetworkModule {
    fun provideOkHttpClient(tokenProvider: TokenProvider): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor(tokenProvider))   // 认证拦截器
            .addInterceptor(LoggingInterceptor())             // 日志拦截器
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .build()
    }

    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/v1/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    fun provideUserApi(retrofit: Retrofit): UserApi = retrofit.create(UserApi::class.java)
}


// ---- 6. 统一错误处理扩展函数 ----
// 把 Retrofit Response 转换成 NetworkResult，集中处理各种异常
suspend fun <T> safeApiCall(apiCall: suspend () -> Response<T>): NetworkResult<T> {
    return try {
        val response = apiCall()
        if (response.isSuccessful) {
            val body = response.body()
            if (body != null) {
                NetworkResult.Success(body)
            } else {
                NetworkResult.Error(-1, "响应体为空")
            }
        } else {
            // HTTP 错误（4xx, 5xx）
            NetworkResult.Error(response.code(), response.message())
        }
    } catch (e: HttpException) {
        NetworkResult.Error(e.code(), e.message())
    } catch (e: IOException) {
        // 网络不通、连接超时
        NetworkResult.NetworkException
    } catch (e: Exception) {
        NetworkResult.Error(-1, e.message ?: "未知错误")
    }
}


// ---- 7. RemoteDataSource 使用 ----
class UserRemoteDataSource(private val api: UserApi) {
    suspend fun getUser(id: Int): NetworkResult<UserDto> = safeApiCall {
        api.getUser(id)   // 直接传入 API 调用，统一处理异常
    }
}


// ---- 8. ViewModel 消费 NetworkResult ----
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<User>>(UiState.Loading)
    val uiState = _uiState.asStateFlow()

    fun loadUser(id: Int) {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            when (val result = repository.getUser(id)) {
                is NetworkResult.Success        -> _uiState.value = UiState.Success(result.data.toUser())
                is NetworkResult.Error          -> _uiState.value = UiState.Error("${result.code}: ${result.message}")
                is NetworkResult.NetworkException -> _uiState.value = UiState.Error("网络不可用，请检查网络连接")
            }
        }
    }
}
```

---

# 三、Kotlin 高级特性（高级岗必问）

---

## Q：内联函数（inline）有什么作用？

**答：**

`inline` 关键字让编译器把函数体直接复制到调用处，消除函数调用开销，特别适合高阶函数（Lambda 参数）。

```kotlin
// ============================================================
// 场景：高阶函数传 Lambda，每次调用都会创建 Lambda 对象（GC 压力）
// ============================================================

// ❌ 普通高阶函数：每次调用都 new 一个 Lambda 对象
fun <T> measureTime(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时：${System.currentTimeMillis() - start}ms")
    return result
}

// ✅ inline：编译时把函数体插入调用处，不创建 Lambda 对象
inline fun <T> measureTimeInline(block: () -> T): T {
    val start = System.currentTimeMillis()
    val result = block()
    println("耗时：${System.currentTimeMillis() - start}ms")
    return result
}

// 调用方式相同，但 inline 版本编译后没有 Lambda 对象创建
val result = measureTimeInline {
    fetchData()
}
// 编译后等价于：
// val start = System.currentTimeMillis()
// val result = fetchData()
// println("耗时：${System.currentTimeMillis() - start}ms")


// ============================================================
// reified：inline 函数中获取泛型的真实类型
// ============================================================

// ❌ 普通泛型：T 在运行时被擦除，无法用 T::class
fun <T> parseJson(json: String): T {
    // return Gson().fromJson(json, T::class.java)  // ❌ 编译错误，T 类型被擦除
}

// ✅ inline + reified：T 的真实类型在运行时可用
inline fun <reified T> parseJson(json: String): T {
    return Gson().fromJson(json, T::class.java)  // ✅ T::class.java 可用
}

// 使用
val user: User = parseJson<User>(jsonString)
val list: List<Article> = parseJson<List<Article>>(jsonString)


// ============================================================
// noinline / crossinline：inline 函数中的 Lambda 控制
// ============================================================

inline fun doSomething(
    crossinline onSuccess: () -> Unit,  // crossinline：不允许在 Lambda 里用 return（非局部返回）
    noinline onError: (Throwable) -> Unit  // noinline：这个 Lambda 不内联（当作对象传递）
) {
    // crossinline 的 Lambda 可以在另一个 Lambda 中调用
    runOnUiThread { onSuccess() }  // ✅ crossinline 允许这样用

    // noinline 的 Lambda 可以存储为变量或传递
    someCallback = onError  // ✅ noinline 允许存储
}
```

---

## Q：扩展函数和扩展属性

**答：**

```kotlin
// ============================================================
// 扩展函数：给已有类添加方法，不修改源码
// ============================================================

// 给 Context 添加 toast 扩展函数
fun Context.toast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

// 给 String 添加有效性检查
fun String?.isValidEmail(): Boolean {
    return !isNullOrEmpty() && android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()
}

// 给 View 添加防抖点击（500ms 内只响应一次）
fun View.setOnSingleClickListener(intervalMs: Long = 500, action: (View) -> Unit) {
    var lastClickTime = 0L
    setOnClickListener { view ->
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastClickTime >= intervalMs) {
            lastClickTime = currentTime
            action(view)
        }
    }
}

// 使用
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        toast("登录成功")                          // 直接用，不需要 Toast.makeText(...)

        val email = "test@example.com"
        if (email.isValidEmail()) { /* ... */ }    // 直观的语法

        button.setOnSingleClickListener {          // 自动防抖
            login()
        }
    }
}


// ============================================================
// 扩展属性：给已有类添加属性
// ============================================================

// 给 Int 添加 dp 转 px 的扩展属性
val Int.dp: Int
    get() = (this * Resources.getSystem().displayMetrics.density).toInt()

val Int.sp: Float
    get() = (this * Resources.getSystem().displayMetrics.scaledDensity)

// 使用
textView.textSize = 16.sp   // 比 TypedValue.applyDimension(...) 简洁多了
view.setPadding(16.dp, 8.dp, 16.dp, 8.dp)


// ============================================================
// 作用域函数：let / run / with / apply / also
// ============================================================

// let：对非 null 值执行操作
val name: String? = getUserName()
name?.let { n ->
    textView.text = n    // n 就是非 null 的 name
    saveToCache(n)
}

// apply：链式配置对象，返回对象本身
val paint = Paint().apply {
    color = Color.RED
    strokeWidth = 5f
    style = Paint.Style.STROKE
    // 不需要每行都写 paint.xxx，直接用属性名
}

// also：执行附加操作，返回对象本身（适合调试日志）
val list = mutableListOf<String>()
    .also { println("创建了列表") }
    .apply { add("item1") }
    .also { println("添加后大小：${it.size}") }

// with：对同一个对象调用多个方法
with(viewBinding) {
    titleTextView.text = article.title
    contentTextView.text = article.content
    authorTextView.text = article.author
    // 不需要每行都写 viewBinding.xxx
}

// run：类似 let，但使用 this 而非 it，常用于对象初始化
val result = StringBuilder().run {
    append("Hello")
    append(" World")
    toString()   // 返回值
}
```

---

## Q：委托属性（by）

**答：**

```kotlin
// ============================================================
// 常用委托属性
// ============================================================

class UserViewModel : ViewModel() {

    // by viewModels()：Activity/Fragment 中获取 ViewModel
    // by lazy：懒加载，第一次访问时才初始化（线程安全）
    private val heavyObject by lazy {
        HeavyObject()  // 只在第一次用到时才创建
    }
}

// by Delegates.observable：值变化时自动回调
var score: Int by Delegates.observable(0) { property, oldValue, newValue ->
    println("分数从 $oldValue 变为 $newValue")
    if (newValue >= 100) showWinDialog()
}

// by Delegates.notNull()：非空委托，未初始化就访问会抛异常
var userId: String by Delegates.notNull()
// userId = "123"  ← 必须赋值后才能访问

// 自定义委托属性（SharedPreferences 存取）
class SharedPrefsDelegate(
    private val prefs: SharedPreferences,
    private val key: String,
    private val default: String = ""
) : ReadWriteProperty<Any, String> {

    override fun getValue(thisRef: Any, property: KProperty<*>): String {
        return prefs.getString(key, default) ?: default
    }

    override fun setValue(thisRef: Any, property: KProperty<*>, value: String) {
        prefs.edit().putString(key, value).apply()
    }
}

// 使用自定义委托：读写 SharedPreferences 像访问普通属性一样
class UserPrefs(prefs: SharedPreferences) {
    var token: String by SharedPrefsDelegate(prefs, "token")
    var username: String by SharedPrefsDelegate(prefs, "username")
}

val prefs = UserPrefs(getSharedPreferences("user", MODE_PRIVATE))
prefs.token = "abc123"    // 自动写入 SharedPreferences
val t = prefs.token       // 自动从 SharedPreferences 读取
```

---

# 四、高级 Android 话题

---

## Q：Jetpack Compose 与 View 系统对比，各自适用场景？

**答：**

| 对比维度 | 传统 View 系统 | Jetpack Compose |
|----------|--------------|----------------|
| 编程范式 | 命令式（告诉系统怎么做）| 声明式（描述 UI 应该是什么样）|
| 布局描述 | XML + Kotlin 分离 | 纯 Kotlin |
| 状态更新 | 手动 `setText`、`setVisibility` | 状态变化自动触发重组 |
| 学习曲线 | 低（生态成熟，资料多）| 中（需要转变思维方式）|
| 互操作 | — | 可嵌入传统 View，也可在 Compose 中嵌入 View |
| Google 推荐 | 维护期，不新增功能 | 主推，新功能优先 Compose |

```kotlin
// ============================================================
// Compose 与 View 互操作（迁移期必用）
// ============================================================

// 1. 在传统 Activity/Fragment 中嵌入 Compose UI
class ArticleFragment : Fragment() {
    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
        // 返回 ComposeView，在里面写 Compose 代码
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
            setContent {
                MaterialTheme {
                    ArticleScreen()  // Composable 函数
                }
            }
        }
    }
}

// 2. 在 Compose 中嵌入传统 View（如 MapView）
@Composable
fun MapScreen() {
    AndroidView(
        factory = { context ->
            // 创建传统 View（这里是地图）
            MapboxMapView(context).apply {
                // 初始化地图
            }
        },
        update = { mapView ->
            // 当 Compose 状态变化时，更新 View
        }
    )
}
```

---

## Q：Hilt 依赖注入是什么？为什么要用？

**答：**

依赖注入（DI）是一种设计模式：对象不自己创建依赖，而是从外部接收。Hilt 是 Google 为 Android 封装的 Dagger2，简化了依赖注入的配置。

**好处：**
1. 解耦：`ViewModel` 不需要知道 `Repository` 怎么创建的
2. 可测试：测试时可以替换为 Mock 依赖
3. 单例管理：同一个 Scope 内的对象自动复用

```kotlin
// ============================================================
// Hilt 完整示例
// ============================================================

// 第一步：Application 标记
@HiltAndroidApp
class MyApplication : Application()


// 第二步：定义如何提供依赖（Module）
@Module
@InstallIn(SingletonComponent::class)   // 整个 App 生命周期内单例
object NetworkModule {

    @Provides
    @Singleton   // 只创建一次
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(LoggingInterceptor())
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideUserApi(retrofit: Retrofit): UserApi = retrofit.create(UserApi::class.java)
}


// 第三步：DataSource 和 Repository（构造器注入）
class UserRemoteDataSource @Inject constructor(
    private val api: UserApi  // Hilt 自动注入 UserApi
)

class UserRepository @Inject constructor(
    private val remote: UserRemoteDataSource
)


// 第四步：ViewModel（Hilt 自动注入）
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository  // Hilt 自动注入 Repository
) : ViewModel()


// 第五步：Activity/Fragment（使用注入）
@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    // by viewModels()：Hilt 自动创建 ViewModel 并注入依赖
    private val viewModel: UserViewModel by viewModels()
}
// 整个链路：Activity → ViewModel → Repository → DataSource → Api
// 全部由 Hilt 自动管理，不需要手动 new
```

---

## Q：Room 数据库使用和优化？

**答：**

```kotlin
// ============================================================
// Room 完整示例（健康数据本地存储）
// ============================================================

// 1. Entity（表结构定义）
@Entity(tableName = "health_records")
data class HealthRecord(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val userId: String,
    val heartRate: Int,           // 心率（bpm）
    val steps: Int,               // 步数
    val timestamp: Long,          // 时间戳
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)


// 2. DAO（数据访问对象）
@Dao
interface HealthRecordDao {

    // suspend：在协程中使用，不阻塞主线程
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(record: HealthRecord): Long

    @Insert
    suspend fun insertAll(records: List<HealthRecord>)

    @Update
    suspend fun update(record: HealthRecord)

    @Delete
    suspend fun delete(record: HealthRecord)

    // 查询最近 7 天的数据
    @Query("SELECT * FROM health_records WHERE userId = :userId AND timestamp >= :startTime ORDER BY timestamp DESC")
    suspend fun getRecords(userId: String, startTime: Long): List<HealthRecord>

    // 返回 Flow：数据库有变化时自动推送新数据（配合 StateFlow 使用）
    @Query("SELECT * FROM health_records WHERE userId = :userId ORDER BY timestamp DESC LIMIT 1")
    fun observeLatest(userId: String): Flow<HealthRecord?>

    // 聚合查询：计算某段时间的平均心率
    @Query("SELECT AVG(heartRate) FROM health_records WHERE userId = :userId AND timestamp BETWEEN :start AND :end")
    suspend fun getAverageHeartRate(userId: String, start: Long, end: Long): Float?

    // 分页查询（配合 Paging3）
    @Query("SELECT * FROM health_records WHERE userId = :userId ORDER BY timestamp DESC")
    fun getRecordsPaged(userId: String): PagingSource<Int, HealthRecord>
}


// 3. Database（数据库定义）
@Database(
    entities = [HealthRecord::class],
    version = 2,                         // 版本号，升级时递增
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun healthRecordDao(): HealthRecordDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2)   // 数据库迁移
                .build()
                .also { INSTANCE = it }
            }
        }
    }
}


// 4. 数据库迁移（版本升级时不丢数据）
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // 版本 1→2：给 health_records 表新增 steps 列
        database.execSQL("ALTER TABLE health_records ADD COLUMN steps INTEGER NOT NULL DEFAULT 0")
    }
}


// 5. 在 Repository 中使用
class HealthRepository(private val dao: HealthRecordDao) {

    // 数据库有数据时直接返回，通过 Flow 监听变化
    fun observeLatestHeartRate(userId: String): Flow<Int?> {
        return dao.observeLatest(userId).map { it?.heartRate }
    }

    suspend fun saveRecord(record: HealthRecord) {
        withContext(Dispatchers.IO) {
            dao.insert(record)
        }
    }

    suspend fun getWeeklyAverage(userId: String): Float? {
        val weekAgo = System.currentTimeMillis() - 7 * 24 * 60 * 60 * 1000L
        return withContext(Dispatchers.IO) {
            dao.getAverageHeartRate(userId, weekAgo, System.currentTimeMillis())
        }
    }
}
```

---

# 五、算法题（LeetCode 中等难度，面试高频）

---

## 算法1：反转链表（简单，基础必会）

```kotlin
class ListNode(var `val`: Int) {
    var next: ListNode? = null
}

// 迭代法：时间 O(n)，空间 O(1)
fun reverseList(head: ListNode?): ListNode? {
    var prev: ListNode? = null  // 前一个节点
    var curr = head             // 当前节点

    while (curr != null) {
        val next = curr.next    // 保存下一个节点
        curr.next = prev        // 反转指针
        prev = curr             // prev 前进
        curr = next             // curr 前进
    }

    return prev  // prev 就是新的头节点
}

// 例：1 → 2 → 3 → null
// 第1步：prev=null, curr=1  →  1.next=null, prev=1, curr=2
// 第2步：prev=1,    curr=2  →  2.next=1,    prev=2, curr=3
// 第3步：prev=2,    curr=3  →  3.next=2,    prev=3, curr=null
// 结果：3 → 2 → 1 → null，返回 prev=3
```

---

## 算法2：二叉树层序遍历（中等，BFS）

```kotlin
class TreeNode(var `val`: Int) {
    var left: TreeNode? = null
    var right: TreeNode? = null
}

// 层序遍历：返回每层的节点值
// BFS（广度优先），用队列
fun levelOrder(root: TreeNode?): List<List<Int>> {
    val result = mutableListOf<List<Int>>()
    if (root == null) return result

    val queue: ArrayDeque<TreeNode> = ArrayDeque()
    queue.add(root)

    while (queue.isNotEmpty()) {
        val levelSize = queue.size      // 当前层的节点数
        val currentLevel = mutableListOf<Int>()

        // 处理当前层的所有节点
        repeat(levelSize) {
            val node = queue.removeFirst()
            currentLevel.add(node.`val`)

            // 把下一层节点加入队列
            node.left?.let { queue.add(it) }
            node.right?.let { queue.add(it) }
        }

        result.add(currentLevel)  // 当前层处理完，加入结果
    }

    return result
}

// 例：
//     1
//    / \
//   2   3
//  / \
// 4   5
// 结果：[[1], [2, 3], [4, 5]]
```

---

## 算法3：两数之和（简单，HashMap）

```kotlin
// 暴力 O(n²) 不说了，用 HashMap 优化到 O(n)
fun twoSum(nums: IntArray, target: Int): IntArray {
    // key：数字值，value：数字的下标
    val map = HashMap<Int, Int>()

    for (i in nums.indices) {
        val complement = target - nums[i]   // 需要找的另一个数

        if (map.containsKey(complement)) {
            return intArrayOf(map[complement]!!, i)  // 找到了
        }

        map[nums[i]] = i   // 没找到，把当前数存入 map
    }

    return intArrayOf()  // 题目保证有解，这里不会到达
}

// 例：nums = [2, 7, 11, 15], target = 9
// i=0: complement=7，map 没有 7，存入 {2:0}
// i=1: complement=2，map 有 2（下标0），返回 [0, 1]
```

---

## 算法4：爬楼梯（简单，动态规划）

```kotlin
// n 阶楼梯，每次可以走 1 步或 2 步，有多少种走法？
// 规律：f(n) = f(n-1) + f(n-2)（斐波那契数列）

fun climbStairs(n: Int): Int {
    if (n <= 2) return n

    var prev2 = 1   // f(1) = 1
    var prev1 = 2   // f(2) = 2

    for (i in 3..n) {
        val curr = prev1 + prev2  // f(n) = f(n-1) + f(n-2)
        prev2 = prev1
        prev1 = curr
    }

    return prev1

    // 滚动数组优化：只存前两个值，空间 O(1)
}

// n=1: 1 种 [1]
// n=2: 2 种 [1+1, 2]
// n=3: 3 种 [1+1+1, 1+2, 2+1]
// n=4: 5 种 = f(3) + f(2) = 3 + 2
```

---

## 算法5：最长无重复子串（中等，滑动窗口）

```kotlin
// 给一个字符串，找最长的无重复字符的子串长度
fun lengthOfLongestSubstring(s: String): Int {
    val charIndex = HashMap<Char, Int>()  // 记录每个字符最近出现的位置
    var maxLen = 0
    var left = 0  // 滑动窗口左边界

    for (right in s.indices) {
        val ch = s[right]

        // 如果这个字符出现过，且在窗口内，移动左边界（跳过重复的）
        if (charIndex.containsKey(ch) && charIndex[ch]!! >= left) {
            left = charIndex[ch]!! + 1
        }

        charIndex[ch] = right              // 更新字符最近出现的位置
        maxLen = maxOf(maxLen, right - left + 1)  // 更新最大长度
    }

    return maxLen
}

// 例："abcabcbb"
// right=0(a): left=0, window="a",    maxLen=1
// right=1(b): left=0, window="ab",   maxLen=2
// right=2(c): left=0, window="abc",  maxLen=3
// right=3(a): a在0，left移到1, window="bca", maxLen=3
// right=4(b): b在1，left移到2, window="cab", maxLen=3
// right=5(c): c在2，left移到3, window="abc", maxLen=3
// 结果：3
```

---

## 算法6：合并两个有序链表（简单，递归/迭代）

```kotlin
// 迭代法
fun mergeTwoLists(list1: ListNode?, list2: ListNode?): ListNode? {
    // 哑节点（dummy）技巧：简化边界处理，不需要特判头节点
    val dummy = ListNode(0)
    var curr = dummy

    var l1 = list1
    var l2 = list2

    while (l1 != null && l2 != null) {
        if (l1.`val` <= l2.`val`) {
            curr.next = l1   // 取 l1 的节点
            l1 = l1.next
        } else {
            curr.next = l2   // 取 l2 的节点
            l2 = l2.next
        }
        curr = curr.next!!
    }

    // 把剩余的（较长的链表）直接接上
    curr.next = l1 ?: l2

    return dummy.next  // dummy.next 就是真正的头节点
}
```

---

# 六、技术选型常见追问

---

## Q：为什么选 Glide 而不是 Picasso？

| 对比 | Glide | Picasso |
|------|-------|---------|
| GIF 支持 | ✅ 原生支持 | ❌ 不支持 |
| 内存缓存 | 默认 RGB_565（内存小）| 默认 ARGB_8888（内存大）|
| 生命周期 | 自动感知 Activity/Fragment 生命周期 | 需要手动管理 |
| 视频缩略图 | ✅ 支持 | ❌ 不支持 |
| 社区活跃 | 更活跃，Google 推荐 | 较少更新 |

**结论：** 移动端一般选 Glide，功能更全、内存管理更好。

---

## Q：为什么 MVVM 比 MVP 好？

| 对比 | MVP | MVVM |
|------|-----|------|
| View 与逻辑耦合 | Presenter 持有 View 接口，仍有一定耦合 | ViewModel 完全不引用 View |
| 旋转屏幕 | Presenter 随 Activity 销毁重建，需重新初始化 | ViewModel 存活，数据不丢失 |
| 测试性 | 需要 Mock View 接口 | ViewModel 完全不依赖 Android，纯 JVM 单元测试 |
| 数据驱动 | 手动调用 `view.showData()` | 数据变化自动驱动 UI 更新（Flow/LiveData）|

**结论：** MVVM + Kotlin Coroutines/Flow 是 Google 官方推荐的最佳实践，新项目首选。

---

## Q：OkHttp 的拦截器链原理？

```kotlin
// ============================================================
// 拦截器链：责任链模式
// ============================================================

// OkHttp 的拦截器按顺序组成链：
// 自定义拦截器1 → 自定义拦截器2 → ... → 网络拦截器 → 真实网络请求

// 每个拦截器可以：
// 1. 在请求发出前处理（修改 Request）
// 2. 把请求传递给下一个拦截器（chain.proceed()）
// 3. 在收到响应后处理（修改 Response）

class RetryInterceptor(private val maxRetry: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var response: Response? = null

        while (attempt < maxRetry) {
            try {
                response = chain.proceed(chain.request())  // 传递给下一个拦截器（或发送请求）
                if (response.isSuccessful) return response  // 成功则直接返回
            } catch (e: IOException) {
                attempt++
                if (attempt >= maxRetry) throw e  // 重试次数用完，抛出异常
                Thread.sleep(1000L * attempt)     // 指数退避（1s, 2s, 3s...）
            }
            response?.close()
            attempt++
        }

        return response!!
    }
}

// 添加到 OkHttpClient
OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor())    // 先加的先执行
    .addInterceptor(LoggingInterceptor())
    .addInterceptor(RetryInterceptor())   // 后加的后执行
    .build()

// 请求时执行顺序：Auth → Logging → Retry → 网络
// 响应时执行顺序：网络 → Retry → Logging → Auth（逆序）
```

---

*文件生成时间：2026-03-05 | 第三优先级：管理岗 / 高级工程师*
