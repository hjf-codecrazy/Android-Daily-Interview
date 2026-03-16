# Android 完整学习笔记 - Part 4

> 包含详细答案 + 代码示例，适合深度学习和理解
>
> 共 40 题：Kotlin（20题）+ 数据结构（9题）+ 设计模式（11题）

---

## 📌 Kotlin

### 1. Kotlin 的基本特性

#### 详细答案

| 特性 | 说明 |
|------|------|
| **空安全** | 编译期区分可空/非空类型，避免 NPE |
| **扩展函数** | 不修改类即可添加新函数 |
| **数据类** | data class 自动生成 equals/hashCode/toString/copy |
| **协程** | 轻量级并发，简化异步代码 |
| **函数式** | Lambda、高阶函数、内联函数 |
| **类型推断** | val/var 自动推断类型 |

#### 代码示例

```kotlin
// 1. 空安全
fun nullSafety() {
    var name: String = "Android"   // 非空，不能赋值 null
    var nullableName: String? = null  // 可空

    // 安全调用 ?.
    val length = nullableName?.length  // null 则返回 null

    // Elvis 运算符 ?:
    val len = nullableName?.length ?: 0  // null 则返回默认值 0

    // 非空断言 !!（慎用，null 时抛 NPE）
    val forceLen = nullableName!!.length

    // let 配合 ?. 使用
    nullableName?.let { n ->
        println("名字是: $n")  // 只有非空才执行
    }

    // 安全类型转换
    val obj: Any = "Hello"
    val str: String? = obj as? String  // 失败返回 null（不抛异常）
}

// 2. 数据类
data class User(
    val id: Long,
    val name: String,
    val email: String,
    val age: Int = 18  // 默认参数
)

fun dataClassDemo() {
    val user1 = User(1, "张三", "zhangsan@example.com")
    val user2 = User(1, "张三", "zhangsan@example.com")

    println(user1 == user2)    // true（比较内容）
    println(user1.toString())  // User(id=1, name=张三, email=...)

    // copy 创建副本（只改部分字段）
    val user3 = user1.copy(name = "李四", age = 25)

    // 解构
    val (id, name, email) = user1
    println("$id $name $email")
}

// 3. 扩展函数
fun String.isPalindrome(): Boolean = this == this.reversed()

fun Int.isEven(): Boolean = this % 2 == 0

fun List<Int>.sum(): Int = fold(0) { acc, i -> acc + i }

fun extensionDemo() {
    println("racecar".isPalindrome())  // true
    println(4.isEven())                // true
    println(listOf(1, 2, 3).sum())     // 6
}

// 4. when 表达式（比 switch 强大）
fun describeNumber(x: Int): String = when {
    x < 0  -> "负数"
    x == 0 -> "零"
    x < 10 -> "个位数"
    else   -> "大数"
}

fun describeType(obj: Any): String = when (obj) {
    is Int    -> "整数: $obj"
    is String -> "字符串长度: ${obj.length}"
    is List<*> -> "列表大小: ${obj.size}"
    else      -> "未知类型"
}
```

#### 核心要点

```
Kotlin vs Java：
├─ 空安全：? 标注可空，?. 安全调用，?: Elvis
├─ val/var：val 不可变（final），var 可变
├─ data class：自动生成 equals/hashCode/copy/toString
├─ 扩展函数：为任何类添加方法（不需要继承）
└─ when：比 switch 更强大，可匹配类型/条件/范围
```

---

### 2. Kotlin 协程（Coroutines）

#### 详细答案

协程是**轻量级的并发单元**，可以挂起（suspend）而不阻塞线程。

**与线程的区别：**
- 线程：操作系统级别，创建销毁开销大，切换需要内核参与
- 协程：用户空间，创建极轻量（可创建成千上万个），挂起/恢复由运行时控制

#### 代码示例

```kotlin
// 1. 基本协程
fun basicCoroutine() {
    // CoroutineScope 定义协程生命周期
    val scope = CoroutineScope(Dispatchers.Main)

    scope.launch {
        // 在主线程启动协程
        val result = withContext(Dispatchers.IO) {
            // 切换到 IO 线程执行耗时操作
            fetchDataFromNetwork()
        }
        // 自动切回主线程更新 UI
        updateUI(result)
    }
}

// 2. suspend 函数
suspend fun fetchDataFromNetwork(): String {
    delay(1000)  // 挂起 1 秒（不阻塞线程）
    return "数据加载完成"
}

// 3. async/await（并行执行）
suspend fun loadPageData(): PageData {
    return coroutineScope {
        val user = async(Dispatchers.IO) { fetchUser() }
        val news = async(Dispatchers.IO) { fetchNews() }
        // 等待两个任务都完成
        PageData(user.await(), news.await())
    }
}

// 4. 调度器（Dispatchers）
fun dispatchersDemo() {
    CoroutineScope(SupervisorJob()).launch {
        // Dispatchers.Main：主线程（UI 操作）
        withContext(Dispatchers.Main) { updateUI() }

        // Dispatchers.IO：IO 操作（网络、文件、数据库）
        withContext(Dispatchers.IO) { readFile() }

        // Dispatchers.Default：CPU 密集型（排序、计算）
        withContext(Dispatchers.Default) { heavyComputation() }
    }
}

// 5. Flow（冷流，响应式数据流）
fun flowDemo(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)  // 发射值
    }
}

suspend fun collectFlow() {
    flowDemo()
        .map { it * 2 }      // 转换
        .filter { it > 4 }   // 过滤
        .catch { e -> println("错误: $e") }  // 异常处理
        .collect { value ->  // 收集
            println(value)
        }
}

// 6. StateFlow 和 SharedFlow
class FlowViewModel : ViewModel() {
    // StateFlow：有状态，新订阅者立即收到最新值
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    // SharedFlow：事件流，可控制缓存和重放
    private val _events = MutableSharedFlow<String>()
    val events: SharedFlow<String> = _events.asSharedFlow()

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = fetchData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "")
            }
        }
    }

    fun sendEvent(message: String) {
        viewModelScope.launch {
            _events.emit(message)
        }
    }
}

// 7. 协程异常处理
fun exceptionHandling() {
    // CoroutineExceptionHandler（捕获未处理的异常）
    val handler = CoroutineExceptionHandler { _, exception ->
        Log.e("Coroutine", "异常: ${exception.message}")
    }

    CoroutineScope(Dispatchers.Main + handler).launch {
        // 方式 1：try-catch
        try {
            riskyOperation()
        } catch (e: Exception) {
            println("捕获: ${e.message}")
        }
    }

    // SupervisorJob：子协程异常不影响其他子协程
    val supervisorScope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    supervisorScope.launch {
        launch { throw Exception("子协程1失败") }  // 不影响下面的协程
        launch { delay(1000); println("子协程2正常完成") }
    }
}
```

#### 核心要点

```
协程核心概念：
├─ suspend 函数：可以挂起（不阻塞线程）
├─ launch：启动不返回结果的协程（Fire and Forget）
├─ async/await：启动返回 Deferred 结果的协程
└─ withContext：切换执行上下文（线程调度器）

Flow vs LiveData：
├─ Flow：冷流，订阅时才开始执行，适合数据流
├─ StateFlow：热流，有状态，类似 LiveData
└─ SharedFlow：热流，适合一次性事件

异常处理优先级：
try-catch > CoroutineExceptionHandler > 崩溃
```

---

### 3. Kotlin 高阶函数和 Lambda

#### 详细答案

**高阶函数**：接受函数作为参数，或返回一个函数的函数。

#### 代码示例

```kotlin
// 1. 高阶函数
fun calculate(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}

fun higherOrderDemo() {
    val sum = calculate(3, 4) { a, b -> a + b }     // 7
    val product = calculate(3, 4) { a, b -> a * b } // 12
    println("$sum $product")
}

// 2. 常用高阶函数
fun collectionOps() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

    // map：转换每个元素
    val doubled = numbers.map { it * 2 }

    // filter：过滤
    val evens = numbers.filter { it % 2 == 0 }

    // reduce/fold：聚合
    val sum = numbers.reduce { acc, i -> acc + i }
    val sumWithInit = numbers.fold(100) { acc, i -> acc + i }  // 初始值 100

    // flatMap：展平
    val nested = listOf(listOf(1, 2), listOf(3, 4))
    val flat = nested.flatMap { it }  // [1, 2, 3, 4]

    // groupBy：分组
    val grouped = numbers.groupBy { if (it % 2 == 0) "偶数" else "奇数" }

    // sortedBy：排序
    val users = listOf(User(1, "张三", "", 20), User(2, "李四", "", 18))
    val sorted = users.sortedBy { it.age }

    // any/all/none：判断
    val hasEven = numbers.any { it % 2 == 0 }
    val allPositive = numbers.all { it > 0 }
    val noneNegative = numbers.none { it < 0 }
}

// 3. 作用域函数
fun scopeFunctions() {
    val user = User(1, "张三", "zhangsan@example.com")

    // let：对象转换，返回 Lambda 结果
    val name = user.let { "姓名: ${it.name}" }

    // also：附加操作，返回对象本身
    val user2 = user.also { println("处理用户: ${it.name}") }

    // apply：配置对象，返回对象本身（常用于 Builder 模式）
    val intent = Intent().apply {
        action = Intent.ACTION_VIEW
        putExtra("userId", "123")
    }

    // run：执行代码块，返回 Lambda 结果
    val length = user.run { name.length + email.length }

    // with：非扩展函数，对对象执行多个操作
    val info = with(user) {
        "id=$id, name=$name, email=$email"
    }
}

// 4. inline 内联函数（消除 Lambda 开销）
inline fun measureTime(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

// noinline：部分参数不内联
inline fun inlineDemo(noinline callback: () -> Unit) {
    // callback 不会被内联（可以存储）
    storeCallback(callback)
}

// crossinline：禁止非局部返回
inline fun runSafely(crossinline block: () -> Unit) {
    try {
        block()
    } catch (e: Exception) {
        e.printStackTrace()
    }
}
```

#### 核心要点

```
作用域函数选择：
├─ let：对可空对象操作，或将对象转换为结果
├─ apply：配置对象属性（Builder 风格）
├─ also：附加副作用（日志、调试）
├─ run：对对象执行代码块并返回结果
└─ with：对非可空对象执行多个操作

inline 函数：
├─ 将 Lambda 代码内联到调用处（减少函数对象开销）
├─ 支持非局部 return（从外层函数返回）
└─ 适用于频繁调用的工具函数
```

---

### 4. Kotlin 密封类（Sealed Class）

#### 详细答案

密封类限制子类只能在**同一文件**中定义，非常适合表示**有限状态集合**。

#### 代码示例

```kotlin
// 1. 密封类定义状态
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: Int = -1) : Result<Nothing>()
    object Loading : Result<Nothing>()
    object Empty : Result<Nothing>()
}

// 2. when 配合密封类（穷举检查）
fun handleResult(result: Result<User>) {
    when (result) {
        is Result.Success -> showUser(result.data)
        is Result.Error   -> showError(result.message)
        is Result.Loading -> showLoading()
        is Result.Empty   -> showEmpty()
        // 不需要 else，编译器保证穷举
    }
}

// 3. 密封接口（Kotlin 1.5+）
sealed interface UiEvent {
    object Refresh : UiEvent
    data class Navigate(val route: String) : UiEvent
    data class ShowToast(val message: String) : UiEvent
}

// 4. 实际应用：网络请求结果
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class HttpError(val code: Int, val message: String) : NetworkResult<Nothing>()
    data class NetworkError(val exception: IOException) : NetworkResult<Nothing>()
    object Timeout : NetworkResult<Nothing>()
}

suspend fun <T> safeApiCall(apiCall: suspend () -> T): NetworkResult<T> {
    return try {
        NetworkResult.Success(apiCall())
    } catch (e: HttpException) {
        NetworkResult.HttpError(e.code(), e.message())
    } catch (e: IOException) {
        NetworkResult.NetworkError(e)
    } catch (e: TimeoutCancellationException) {
        NetworkResult.Timeout
    }
}
```

#### 核心要点

```
sealed class vs enum：
├─ enum：每个成员是同一类型，只能有固定字段
└─ sealed class：每个子类可以有不同字段和类型

sealed class vs abstract class：
├─ abstract class：子类可以在任何地方定义
└─ sealed class：子类只能在同一文件中定义（编译器可穷举）

典型使用场景：
├─ UI 状态（Loading/Success/Error）
├─ 网络结果封装
└─ 事件（UiEvent/SideEffect）
```

---

### 5. Kotlin 委托属性

#### 详细答案

委托属性通过 `by` 关键字将属性的 get/set 逻辑委托给另一个对象。

#### 代码示例

```kotlin
// 1. lazy 懒加载（最常用）
class HeavyObject {
    // 第一次访问时才初始化，线程安全
    val heavyResource: List<String> by lazy {
        println("初始化中...")
        loadHeavyData()  // 只执行一次
    }

    // 非线程安全的 lazy（性能更好）
    val fastLazy by lazy(LazyThreadSafetyMode.NONE) {
        "fast"
    }
}

// 2. observable（观察属性变化）
class UserModel {
    var name: String by Delegates.observable("初始值") { property, oldValue, newValue ->
        println("${property.name}: $oldValue → $newValue")
        // 每次赋值都会触发
    }

    // vetoable：可以拒绝变更
    var age: Int by Delegates.vetoable(0) { _, _, newValue ->
        newValue >= 0  // 返回 true 才接受新值
    }
}

// 3. map 委托（从 Map 中读取属性）
class Config(map: Map<String, Any?>) {
    val host: String by map      // map["host"]
    val port: Int by map         // map["port"]
    val debug: Boolean by map    // map["debug"]
}

fun mapDelegateDemo() {
    val config = Config(mapOf(
        "host" to "localhost",
        "port" to 8080,
        "debug" to true
    ))
    println("${config.host}:${config.port}")
}

// 4. 自定义委托
class SharedPrefsDelegate(
    private val prefs: SharedPreferences,
    private val key: String,
    private val default: String = ""
) : ReadWriteProperty<Any?, String> {

    override fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return prefs.getString(key, default) ?: default
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        prefs.edit().putString(key, value).apply()
    }
}

// 使用自定义委托
class UserPrefs(prefs: SharedPreferences) {
    var token: String by SharedPrefsDelegate(prefs, "token")
    var userId: String by SharedPrefsDelegate(prefs, "userId")
}
```

#### 核心要点

```
常用内置委托：
├─ lazy { }：懒初始化，只执行一次
├─ observable { }：属性变化时回调
├─ vetoable { }：可拒绝变更
└─ by map：从 Map 中映射属性

应用场景：
├─ lazy：延迟初始化重量级对象
├─ observable：ViewModel 中追踪属性变化
└─ 自定义委托：SharedPreferences、DataStore 封装
```

---

### 6. Kotlin 泛型和型变

#### 详细答案

| 关键字 | 含义 | Java 等价 | 说明 |
|--------|------|----------|------|
| `out T` | 协变（Covariant） | `? extends T` | 只能读，不能写 |
| `in T` | 逆变（Contravariant） | `? super T` | 只能写，不能读 |
| `T` | 不变 | `T` | 既能读又能写 |

#### 代码示例

```kotlin
// 1. 协变 out（只读，Producer）
class Box<out T>(val value: T) {
    fun get(): T = value
    // fun set(v: T) {}  // ❌ out 不能作为参数类型
}

fun covariantDemo() {
    val intBox: Box<Int> = Box(42)
    val anyBox: Box<Any> = intBox  // ✓ Box<Int> 是 Box<Any> 的子类型
}

// 2. 逆变 in（只写，Consumer）
class Printer<in T> {
    fun print(value: T) { println(value) }
    // fun get(): T {}  // ❌ in 不能作为返回类型
}

fun contravariantDemo() {
    val anyPrinter: Printer<Any> = Printer()
    val stringPrinter: Printer<String> = anyPrinter  // ✓ Printer<Any> 是 Printer<String> 的子类型
}

// 3. 星号投影（类似 Java 的 ?）
fun printList(list: List<*>) {  // 不知道元素类型
    list.forEach { println(it) }
}

// 4. reified 实化类型参数（内联函数中获取泛型类型）
inline fun <reified T> Any.isInstanceOf(): Boolean = this is T

inline fun <reified T> Context.startActivity() {
    startActivity(Intent(this, T::class.java))
}

// 使用
fun reifiedDemo(context: Context) {
    "hello".isInstanceOf<String>()  // true
    42.isInstanceOf<String>()       // false
    context.startActivity<MainActivity>()  // 不需要 MainActivity::class.java
}

// 5. 泛型约束
fun <T : Comparable<T>> findMax(list: List<T>): T? {
    return list.maxOrNull()
}

// 多个约束
fun <T> processData(data: T) where T : Serializable, T : Comparable<T> {
    // T 必须同时实现 Serializable 和 Comparable
}
```

#### 核心要点

```
型变规则（PECS 原则）：
├─ Producer → out（只产生数据，协变）
└─ Consumer → in（只消费数据，逆变）

记忆方式：
├─ out T = List<out T> → 可以读，不能写（生产者）
└─ in T = Comparator<in T> → 可以写，不能读（消费者）

reified 使用场景：
├─ 获取泛型的 Class 对象
├─ is/as 类型检查
└─ 只能在 inline 函数中使用
```

---

### 7. Kotlin 数据类 vs 普通类

#### 详细答案

`data class` 会自动生成：
- `equals()` / `hashCode()`（基于所有主构造器属性）
- `toString()`
- `copy()`
- `componentN()` 函数（支持解构）

#### 代码示例

```kotlin
// 1. data class 特性
data class Point(val x: Int, val y: Int)

fun dataClassFeatures() {
    val p1 = Point(1, 2)
    val p2 = Point(1, 2)
    val p3 = Point(3, 4)

    // equals 比较内容（不是引用）
    println(p1 == p2)   // true
    println(p1 === p2)  // false（不同对象）

    // copy 浅拷贝
    val p4 = p1.copy(x = 10)  // Point(10, 2)

    // 解构
    val (x, y) = p1
    println("x=$x, y=$y")

    // componentN
    println(p1.component1())  // 1
    println(p1.component2())  // 2

    // 在 Map 中正确用作 Key（因为 hashCode 正确实现）
    val map = mapOf(p1 to "原点附近")
    println(map[p2])  // "原点附近"（p1 == p2，hashCode 相同）
}

// 2. data class 注意事项
data class Node(
    val value: Int,
    var next: Node? = null  // ⚠️ var 属性也会参与 equals/hashCode
)

// 3. 继承限制
open class Shape
data class Circle(val radius: Double) : Shape()  // ✓ 可以继承

// data class 不能被继承
// class Sphere : Circle(1.0)  // ❌

// 4. 普通类 vs data class
class NormalUser(val name: String, val age: Int) {
    // 不会自动生成 equals/hashCode/toString
    // 需要手动实现

    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is NormalUser) return false
        return name == other.name && age == other.age
    }

    override fun hashCode(): Int = Objects.hash(name, age)

    override fun toString(): String = "NormalUser(name=$name, age=$age)"
}
```

---

### 8. Kotlin 协程和线程的对比

#### 详细答案

| 对比项 | 线程 | 协程 |
|--------|------|------|
| **创建开销** | 大（~1MB 栈内存） | 极小（几十字节） |
| **数量上限** | 通常几千个 | 可创建数十万个 |
| **切换方式** | 操作系统调度 | 程序控制（挂起/恢复） |
| **阻塞行为** | 阻塞线程 | 挂起协程（不阻塞线程） |
| **并发模型** | 抢占式 | 协作式 |

#### 代码示例

```kotlin
// 线程方式（传统）
fun threadWay() {
    // 创建大量线程开销巨大
    thread {
        val data = fetchFromNetwork()  // 阻塞线程
        runOnUiThread { updateUI(data) }  // 手动切回主线程
    }
}

// 协程方式（现代）
fun coroutineWay() {
    lifecycleScope.launch {
        val data = withContext(Dispatchers.IO) {
            fetchFromNetwork()  // 挂起协程，不阻塞线程
        }
        updateUI(data)  // 自动在主线程执行
    }
}

// 验证协程轻量性
fun coroutineLightWeight() {
    runBlocking {
        // 创建 10 万个协程
        val jobs = (1..100_000).map {
            launch {
                delay(1000)
                print(".")
            }
        }
        jobs.forEach { it.join() }
    }
    // 线程无法做到这一点（会 OOM 或无法创建）
}
```

---

### 9. Kotlin 的 object 关键字

#### 详细答案

`object` 关键字有三种用法：
1. **对象声明**：单例
2. **伴生对象**：类级别的成员（类似 Java 静态成员）
3. **对象表达式**：匿名对象（类似 Java 匿名类）

#### 代码示例

```kotlin
// 1. 对象声明（单例）
object DatabaseManager {
    private val db = createDatabase()

    fun query(sql: String): List<Any> = db.query(sql)
    fun insert(data: Any) = db.insert(data)
}

// 线程安全的单例（Kotlin 保证）
fun singletonDemo() {
    DatabaseManager.query("SELECT * FROM users")
}

// 2. 伴生对象（companion object）
class MyFragment : Fragment() {

    companion object {
        private const val ARG_USER_ID = "userId"

        // 工厂方法（推荐创建 Fragment 的方式）
        fun newInstance(userId: String): MyFragment {
            return MyFragment().apply {
                arguments = bundleOf(ARG_USER_ID to userId)
            }
        }

        // 常量
        const val TAG = "MyFragment"
    }

    private val userId: String by lazy {
        arguments?.getString(ARG_USER_ID) ?: ""
    }
}

// 调用
val fragment = MyFragment.newInstance("123")

// 3. 对象表达式（匿名对象）
fun objectExpression(view: View) {
    // 替代 Java 匿名内部类
    view.setOnClickListener(object : View.OnClickListener {
        override fun onClick(v: View) {
            println("点击了")
        }
    })

    // 更简洁的 Lambda 写法（SAM 接口）
    view.setOnClickListener { println("点击了") }

    // 实现多个接口的匿名对象
    val obj = object : Runnable, Closeable {
        override fun run() { println("running") }
        override fun close() { println("closing") }
    }
}
```

#### 核心要点

```
object 三种用法：
├─ object 声明：单例（线程安全，类加载时初始化）
├─ companion object：伴生对象（工厂方法、常量）
└─ object 表达式：匿名对象（替代匿名内部类）

companion object vs Java static：
├─ companion object 是真正的对象，可以实现接口
└─ 可以通过 @JvmStatic 生成真正的 Java 静态方法
```

---

### 10. Kotlin 中的 inline、noinline、crossinline

#### 详细答案

- **inline**：将函数体和 Lambda 参数都内联到调用处（减少开销，支持非局部 return）
- **noinline**：标记某个 Lambda 参数不内联（可以存储/传递该 Lambda）
- **crossinline**：标记 Lambda 不能使用非局部 return（在间接调用场景中使用）

#### 代码示例

```kotlin
// 1. inline：消除 Lambda 创建开销
inline fun repeat(times: Int, action: () -> Unit) {
    for (i in 0 until times) {
        action()
    }
}
// 调用处：action() 的代码直接被复制到这里，没有 Function 对象创建

// 2. 非局部 return（inline 函数特有）
inline fun forEach(list: List<Int>, action: (Int) -> Unit) {
    for (item in list) {
        action(item)
    }
}

fun findFirst(list: List<Int>): Int? {
    forEach(list) { item ->
        if (item > 5) return item  // 从 findFirst 函数返回（非局部 return）
    }
    return null
}

// 3. noinline：部分 Lambda 不内联
inline fun process(
    inlineBlock: () -> Unit,
    noinline storedBlock: () -> Unit  // 需要存储这个 Lambda
) {
    inlineBlock()  // 内联
    scheduleForLater(storedBlock)  // 存储，不能内联
}

// 4. crossinline：禁止非局部 return
inline fun runAsync(crossinline block: () -> Unit) {
    Thread {
        block()  // 在新线程调用，crossinline 禁止 block 使用非局部 return
    }.start()
}
```

#### 核心要点

```
使用场景：
├─ inline：频繁调用的工具函数（measureTime、run、apply）
├─ noinline：需要把 Lambda 存储到变量或传递给其他函数
└─ crossinline：Lambda 在非直接调用处执行（Thread、Runnable）

内联函数的好处：
├─ 减少 Function 对象创建（每个 Lambda 都是一个对象）
├─ 减少虚函数调用
└─ 支持 reified 类型参数
```

---

### 11. Kotlin 与 Java 互操作注意事项

#### 代码示例

```kotlin
// 1. @JvmStatic - 生成 Java 静态方法
class Utils {
    companion object {
        @JvmStatic
        fun formatDate(date: Long): String = "..."
        // Java 调用：Utils.formatDate(date)
        // 没有 @JvmStatic：Utils.Companion.formatDate(date)
    }
}

// 2. @JvmOverloads - 生成默认参数重载
class Dialog @JvmOverloads constructor(
    context: Context,
    title: String = "",
    message: String = ""
) {
    // Java 可以用 new Dialog(context) 或 new Dialog(context, "标题")
}

// 3. @JvmField - 暴露为 Java 字段（不生成 getter/setter）
class Config {
    @JvmField
    val MAX_RETRY = 3  // Java: Config.MAX_RETRY
}

// 4. @Throws - 声明检查异常（Java 互操作）
@Throws(IOException::class)
fun readFile(path: String): String {
    // Kotlin 没有检查异常，但 Java 需要 try-catch
    return File(path).readText()
}

// 5. Kotlin 调用 Java（注意平台类型）
fun callJava() {
    val javaObj = JavaClass()
    // Java 返回的对象类型是平台类型 T!（可空也可非空）
    // 需要手动处理空安全
    val result: String? = javaObj.getValue()  // 标记为可空
    println(result?.length)
}
```

---

## 📌 数据结构

### 1. 数组和链表的区别

#### 详细答案

| 操作 | 数组 | 链表 |
|------|------|------|
| **随机访问** | O(1) | O(n) |
| **头部插入** | O(n) | O(1) |
| **尾部插入** | O(1)摊销 | O(1)(有尾指针) |
| **中间插入** | O(n) | O(1)(有指针) |
| **查找** | O(n) | O(n) |
| **内存** | 连续，缓存友好 | 非连续，有额外指针开销 |

#### 代码示例

```java
// 手写单链表
public class LinkedList<T> {
    private Node<T> head;
    private int size;

    private static class Node<T> {
        T data;
        Node<T> next;
        Node(T data) { this.data = data; }
    }

    // 头插
    public void addFirst(T data) {
        Node<T> node = new Node<>(data);
        node.next = head;
        head = node;
        size++;
    }

    // 尾插
    public void addLast(T data) {
        Node<T> node = new Node<>(data);
        if (head == null) { head = node; }
        else {
            Node<T> curr = head;
            while (curr.next != null) curr = curr.next;
            curr.next = node;
        }
        size++;
    }

    // 反转链表（迭代）
    public void reverse() {
        Node<T> prev = null;
        Node<T> curr = head;
        while (curr != null) {
            Node<T> next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        head = prev;
    }

    // 判断是否成环（快慢指针）
    public boolean hasCycle() {
        Node<T> slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) return true;
        }
        return false;
    }

    // 查找中间节点（快慢指针）
    public T findMiddle() {
        Node<T> slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
        }
        return slow.data;
    }
}
```

---

### 2. HashMap 的实现原理

#### 详细答案

HashMap 基于**数组 + 链表/红黑树**实现。

**核心流程：**
```
put(key, value)
    ↓
hash(key) → 计算 hash 值
    ↓
index = hash & (capacity - 1)  → 计算数组下标
    ↓
数组[index] 为空 → 直接存放
数组[index] 不为空（Hash 冲突）
    ↓ 链表长度 < 8
    链表（尾插法，JDK 8）
    ↓ 链表长度 >= 8 且数组长度 >= 64
    转换为红黑树
```

**扩容：** 容量 × 2，所有元素重新 hash 分布。

#### 代码示例

```java
// HashMap 核心源码解析（简化版）
public class HashMapDemo {

    public void explainPut() {
        /*
        JDK 8 put 流程：

        1. 计算 hash：
           hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)
           // 高低位异或，让 hash 分布更均匀

        2. 计算下标：
           index = hash & (n - 1)  // n 必须是 2 的幂，等价于 hash % n

        3. 处理冲突：
           - 链表：遍历找相同 key 则更新，否则尾插
           - 链表长度 >= 8 时转红黑树
           - 红黑树长度 <= 6 时退化回链表

        4. 扩容条件：
           size > capacity * loadFactor（默认 0.75）
           新容量 = 旧容量 * 2
        */
    }

    public void importance() {
        /*
        为什么初始容量是 2 的幂？
        index = hash & (n-1) 等价于 hash % n，位运算更快

        为什么负载因子是 0.75？
        时间和空间的权衡：
        - 太小（如 0.5）：频繁扩容，浪费空间
        - 太大（如 1.0）：冲突多，链表/红黑树变长，查询慢

        为什么链表长度 >= 8 转红黑树？
        泊松分布：长度 >= 8 的概率约为千万分之一，
        这种情况下树化带来的查询 O(logn) 比链表 O(n) 明显快
        */
    }
}
```

#### 核心要点

```
HashMap 数据结构：
JDK 7：数组 + 链表
JDK 8：数组 + 链表/红黑树（链表 ≥ 8 转树，≤ 6 转链表）

HashMap vs HashTable：
├─ HashMap：线程不安全，允许 null key/value
└─ HashTable：线程安全（synchronized），不允许 null

HashMap vs ConcurrentHashMap：
├─ HashMap：线程不安全
└─ ConcurrentHashMap：JDK 8 用 CAS + synchronized 分段锁，更高效
```

---

### 3. 二叉树和二叉搜索树

#### 代码示例

```java
public class BinaryTree {
    static class TreeNode {
        int val;
        TreeNode left, right;
        TreeNode(int val) { this.val = val; }
    }

    // 前序遍历（根-左-右）
    public void preOrder(TreeNode root) {
        if (root == null) return;
        System.out.print(root.val + " ");
        preOrder(root.left);
        preOrder(root.right);
    }

    // 中序遍历（左-根-右）：BST 中序遍历得到有序序列
    public void inOrder(TreeNode root) {
        if (root == null) return;
        inOrder(root.left);
        System.out.print(root.val + " ");
        inOrder(root.right);
    }

    // 后序遍历（左-右-根）
    public void postOrder(TreeNode root) {
        if (root == null) return;
        postOrder(root.left);
        postOrder(root.right);
        System.out.print(root.val + " ");
    }

    // 层序遍历（BFS）
    public void levelOrder(TreeNode root) {
        if (root == null) return;
        Queue<TreeNode> queue = new LinkedList<>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            TreeNode node = queue.poll();
            System.out.print(node.val + " ");
            if (node.left != null)  queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
    }

    // 树的最大深度
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;
        return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
    }

    // BST 插入
    public TreeNode insert(TreeNode root, int val) {
        if (root == null) return new TreeNode(val);
        if (val < root.val)      root.left = insert(root.left, val);
        else if (val > root.val) root.right = insert(root.right, val);
        return root;
    }

    // 判断是否是 BST（中序遍历递增）
    private int prev = Integer.MIN_VALUE;
    public boolean isValidBST(TreeNode root) {
        if (root == null) return true;
        if (!isValidBST(root.left)) return false;
        if (root.val <= prev) return false;
        prev = root.val;
        return isValidBST(root.right);
    }
}
```

---

### 4. 排序算法对比

#### 详细答案

| 算法 | 平均时间 | 最坏时间 | 空间 | 稳定 |
|------|---------|---------|------|------|
| 冒泡排序 | O(n²) | O(n²) | O(1) | ✅ |
| 选择排序 | O(n²) | O(n²) | O(1) | ❌ |
| 插入排序 | O(n²) | O(n²) | O(1) | ✅ |
| 归并排序 | O(nlogn) | O(nlogn) | O(n) | ✅ |
| 快速排序 | O(nlogn) | O(n²) | O(logn) | ❌ |
| 堆排序 | O(nlogn) | O(nlogn) | O(1) | ❌ |

#### 代码示例

```java
public class SortAlgorithms {

    // 快速排序
    public void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pivot = partition(arr, low, high);
            quickSort(arr, low, pivot - 1);
            quickSort(arr, pivot + 1, high);
        }
    }

    private int partition(int[] arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1;
        for (int j = low; j < high; j++) {
            if (arr[j] <= pivot) {
                i++;
                swap(arr, i, j);
            }
        }
        swap(arr, i + 1, high);
        return i + 1;
    }

    // 归并排序
    public void mergeSort(int[] arr, int left, int right) {
        if (left < right) {
            int mid = (left + right) / 2;
            mergeSort(arr, left, mid);
            mergeSort(arr, mid + 1, right);
            merge(arr, left, mid, right);
        }
    }

    private void merge(int[] arr, int left, int mid, int right) {
        int[] temp = new int[right - left + 1];
        int i = left, j = mid + 1, k = 0;
        while (i <= mid && j <= right) {
            temp[k++] = (arr[i] <= arr[j]) ? arr[i++] : arr[j++];
        }
        while (i <= mid) temp[k++] = arr[i++];
        while (j <= right) temp[k++] = arr[j++];
        System.arraycopy(temp, 0, arr, left, temp.length);
    }

    private void swap(int[] arr, int i, int j) {
        int t = arr[i]; arr[i] = arr[j]; arr[j] = t;
    }
}
```

---

### 5. 栈和队列

#### 代码示例

```java
// 用两个栈实现队列
public class QueueWithTwoStacks<T> {
    private Stack<T> inStack = new Stack<>();   // 入队栈
    private Stack<T> outStack = new Stack<>();  // 出队栈

    public void enqueue(T item) {
        inStack.push(item);
    }

    public T dequeue() {
        if (outStack.isEmpty()) {
            // 将入队栈全部转移到出队栈
            while (!inStack.isEmpty()) {
                outStack.push(inStack.pop());
            }
        }
        if (outStack.isEmpty()) throw new EmptyStackException();
        return outStack.pop();
    }
}

// 单调栈（用于"下一个更大元素"问题）
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Arrays.fill(result, -1);
    Deque<Integer> stack = new ArrayDeque<>();  // 存下标

    for (int i = 0; i < n; i++) {
        while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
            int idx = stack.pop();
            result[idx] = nums[i];
        }
        stack.push(i);
    }
    return result;
}
```

---

### 6. 红黑树

#### 详细答案

红黑树是**自平衡二叉搜索树**，通过节点颜色（红/黑）和旋转保证树高 O(log n)。

**五条规则：**
1. 每个节点是红色或黑色
2. 根节点是黑色
3. 叶子节点（NIL）是黑色
4. 红色节点的两个子节点都是黑色（不能有连续红色节点）
5. 从任一节点到其叶子节点的所有路径，包含相同数量的黑色节点

**应用场景：**
- Java TreeMap / TreeSet
- Java ConcurrentHashMap（JDK 8 链表树化）
- Linux 进程调度（CFS）

#### 核心要点

```
红黑树 vs AVL 树：
├─ AVL 树：严格平衡（左右高度差 ≤ 1），查询更快
└─ 红黑树：不严格平衡，插入/删除旋转更少，更适合频繁修改

时间复杂度：
├─ 查询：O(log n)
├─ 插入：O(log n)
└─ 删除：O(log n)
```

---

### 7. LruCache 原理

#### 详细答案

LRU（Least Recently Used）最近最少使用，使用**LinkedHashMap** 实现。

#### 代码示例

```java
// 手写 LruCache
public class LruCache<K, V> {
    private final int capacity;
    // LinkedHashMap：accessOrder=true 时按访问顺序排列
    private final LinkedHashMap<K, V> cache;

    public LruCache(int capacity) {
        this.capacity = capacity;
        this.cache = new LinkedHashMap<K, V>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;  // 超出容量自动删除最旧条目
            }
        };
    }

    public synchronized V get(K key) {
        return cache.getOrDefault(key, null);
    }

    public synchronized void put(K key, V value) {
        cache.put(key, value);
    }
}

// Android 中 LruCache 用于图片内存缓存
public class ImageCache {
    private final LruCache<String, Bitmap> memoryCache;

    public ImageCache() {
        // 使用最大内存的 1/8 作为缓存
        int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
        int cacheSize = maxMemory / 8;

        memoryCache = new LruCache<String, Bitmap>(cacheSize) {
            @Override
            protected int sizeOf(String key, Bitmap bitmap) {
                // 返回 Bitmap 占用的 KB 数
                return bitmap.getByteCount() / 1024;
            }
        };
    }

    public void put(String key, Bitmap bitmap) {
        if (get(key) == null) {
            memoryCache.put(key, bitmap);
        }
    }

    public Bitmap get(String key) {
        return memoryCache.get(key);
    }
}
```

---

## 📌 设计模式

### 1. 单例模式（Singleton）

#### 代码示例

```java
// 1. 双重检查锁（DCL）- 推荐
public class Singleton {
    private static volatile Singleton instance;  // volatile 防止指令重排

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {             // 第一次检查（无锁）
            synchronized (Singleton.class) {
                if (instance == null) {     // 第二次检查（有锁）
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// 2. 静态内部类 - 推荐（懒加载 + 线程安全）
public class BetterSingleton {
    private BetterSingleton() {}

    private static class Holder {
        static final BetterSingleton INSTANCE = new BetterSingleton();
    }

    public static BetterSingleton getInstance() {
        return Holder.INSTANCE;
    }
}

// 3. 枚举单例 - 最安全（防止反序列化破坏）
public enum EnumSingleton {
    INSTANCE;
    public void doSomething() {}
}
```

```kotlin
// Kotlin 单例
object AppConfig {
    val apiBaseUrl = "https://api.example.com"
    val maxRetry = 3
}

// 延迟初始化单例
class Repository private constructor() {
    companion object {
        val instance: Repository by lazy { Repository() }
    }
}
```

#### 核心要点

```
单例模式选择：
├─ 饿汉式：类加载就创建，线程安全，无懒加载
├─ DCL：懒加载，需要 volatile 防止指令重排
├─ 静态内部类：懒加载，线程安全，推荐
└─ 枚举：最安全，防反序列化和反射攻击

volatile 的作用（DCL 中）：
└─ 防止 new 操作的指令重排：
   正常顺序：分配内存 → 初始化 → 赋值给引用
   重排后可能：分配内存 → 赋值给引用 → 初始化
   → 另一线程拿到未初始化的对象
```

---

### 2. 观察者模式（Observer）

#### 代码示例

```java
// 经典观察者模式
public interface Observer {
    void update(String event, Object data);
}

public class EventBus {
    private final Map<String, List<Observer>> observers = new HashMap<>();

    public void subscribe(String event, Observer observer) {
        observers.computeIfAbsent(event, k -> new ArrayList<>()).add(observer);
    }

    public void unsubscribe(String event, Observer observer) {
        List<Observer> list = observers.get(event);
        if (list != null) list.remove(observer);
    }

    public void publish(String event, Object data) {
        List<Observer> list = observers.getOrDefault(event, Collections.emptyList());
        for (Observer observer : new ArrayList<>(list)) {
            observer.update(event, data);
        }
    }
}
```

```kotlin
// Kotlin + Flow 实现观察者模式
class EventManager {
    private val _events = MutableSharedFlow<AppEvent>()
    val events = _events.asSharedFlow()

    suspend fun emit(event: AppEvent) = _events.emit(event)
}

sealed class AppEvent {
    data class UserLoggedIn(val user: User) : AppEvent()
    object UserLoggedOut : AppEvent()
    data class NetworkError(val message: String) : AppEvent()
}
```

#### 核心要点

```
观察者模式在 Android 中的应用：
├─ LiveData / Flow（ViewModel → View）
├─ RxJava（响应式编程）
├─ EventBus（全局事件总线）
├─ BroadcastReceiver（系统广播）
└─ ContentObserver（数据变更通知）
```

---

### 3. 工厂模式（Factory）

#### 代码示例

```java
// 简单工厂
public class DialogFactory {
    public static Dialog create(String type, Context context) {
        return switch (type) {
            case "alert"   -> new AlertDialog(context);
            case "confirm" -> new ConfirmDialog(context);
            case "input"   -> new InputDialog(context);
            default        -> throw new IllegalArgumentException("未知类型: " + type);
        };
    }
}

// 抽象工厂（创建相关联的一族对象）
public interface ThemeFactory {
    Button createButton();
    TextView createTextView();
    Color createColor();
}

public class DarkThemeFactory implements ThemeFactory {
    @Override public Button createButton() { return new DarkButton(); }
    @Override public TextView createTextView() { return new DarkTextView(); }
    @Override public Color createColor() { return Color.DARK; }
}

public class LightThemeFactory implements ThemeFactory {
    @Override public Button createButton() { return new LightButton(); }
    @Override public TextView createTextView() { return new LightTextView(); }
    @Override public Color createColor() { return Color.LIGHT; }
}
```

---

### 4. 建造者模式（Builder）

#### 代码示例

```java
// Builder 模式
public class AlertDialog {
    private final String title;
    private final String message;
    private final String positiveText;
    private final String negativeText;
    private final boolean cancelable;

    private AlertDialog(Builder builder) {
        this.title = builder.title;
        this.message = builder.message;
        this.positiveText = builder.positiveText;
        this.negativeText = builder.negativeText;
        this.cancelable = builder.cancelable;
    }

    public static class Builder {
        private String title = "";
        private String message = "";
        private String positiveText = "确定";
        private String negativeText = "取消";
        private boolean cancelable = true;

        public Builder title(String title) { this.title = title; return this; }
        public Builder message(String message) { this.message = message; return this; }
        public Builder positiveText(String text) { this.positiveText = text; return this; }
        public Builder negativeText(String text) { this.negativeText = text; return this; }
        public Builder cancelable(boolean cancelable) { this.cancelable = cancelable; return this; }

        public AlertDialog build() {
            return new AlertDialog(this);
        }
    }
}

// 使用
AlertDialog dialog = new AlertDialog.Builder()
    .title("确认删除")
    .message("该操作不可恢复，确定删除吗？")
    .positiveText("删除")
    .negativeText("取消")
    .cancelable(false)
    .build();
```

---

### 5. 策略模式（Strategy）

#### 代码示例

```kotlin
// 策略接口
interface SortStrategy {
    fun sort(list: MutableList<Int>)
}

class BubbleSort : SortStrategy {
    override fun sort(list: MutableList<Int>) {
        for (i in 0 until list.size - 1)
            for (j in 0 until list.size - 1 - i)
                if (list[j] > list[j + 1]) { val t = list[j]; list[j] = list[j+1]; list[j+1] = t }
    }
}

class QuickSortStrategy : SortStrategy {
    override fun sort(list: MutableList<Int>) = list.sort()
}

// 上下文
class Sorter(private var strategy: SortStrategy) {
    fun setStrategy(strategy: SortStrategy) { this.strategy = strategy }
    fun sort(list: MutableList<Int>) = strategy.sort(list)
}

// 使用
val sorter = Sorter(QuickSortStrategy())
sorter.sort(mutableListOf(3, 1, 4, 1, 5, 9, 2, 6))

// Android 中策略模式应用：
// LayoutManager（LinearLayoutManager / GridLayoutManager / StaggeredGridLayoutManager）
```

---

### 6. 装饰者模式（Decorator）

#### 代码示例

```java
// 装饰者模式（为 InputStream 添加功能）
// Java IO 就是典型的装饰者模式

InputStream raw = new FileInputStream("file.txt");
InputStream buffered = new BufferedInputStream(raw);   // 添加缓冲
DataInputStream data = new DataInputStream(buffered);  // 添加数据读取
// 每一层包装增加功能，不修改原有类

// Android 中的应用：ContextWrapper 装饰 Context
public class ContextWrapper extends Context {
    private Context base;

    public ContextWrapper(Context base) { this.base = base; }

    @Override
    public void startActivity(Intent intent) {
        base.startActivity(intent);  // 委托给被装饰对象
    }
    // 可以重写某些方法，增加额外逻辑
}
// Activity → ContextThemeWrapper → ContextWrapper → Context
```

---

### 7. 代理模式（Proxy）

#### 代码示例

```java
// 静态代理
public interface ImageLoader {
    Bitmap load(String url);
}

public class RealImageLoader implements ImageLoader {
    @Override
    public Bitmap load(String url) {
        return downloadFromNetwork(url);
    }
}

// 代理：添加缓存功能（不修改原类）
public class CachedImageLoader implements ImageLoader {
    private final ImageLoader realLoader;
    private final Map<String, Bitmap> cache = new LruCache<>(100);

    public CachedImageLoader(ImageLoader realLoader) {
        this.realLoader = realLoader;
    }

    @Override
    public Bitmap load(String url) {
        Bitmap cached = cache.get(url);
        if (cached != null) return cached;

        Bitmap bitmap = realLoader.load(url);
        cache.put(url, bitmap);
        return bitmap;
    }
}

// 动态代理（Retrofit 的核心原理）
ImageLoader proxy = (ImageLoader) Proxy.newProxyInstance(
    ImageLoader.class.getClassLoader(),
    new Class[]{ImageLoader.class},
    (proxyObj, method, args) -> {
        System.out.println("调用方法: " + method.getName());
        Object result = method.invoke(realLoader, args);
        System.out.println("方法结束");
        return result;
    }
);
```

---

### 8. 责任链模式（Chain of Responsibility）

#### 代码示例

```kotlin
// OkHttp 拦截器就是责任链模式
abstract class Interceptor {
    var next: Interceptor? = null

    abstract fun intercept(request: Request): Response

    protected fun proceed(request: Request): Response {
        return next?.intercept(request) ?: throw Exception("链末尾没有处理器")
    }
}

class AuthInterceptor : Interceptor() {
    override fun intercept(request: Request): Response {
        val authedRequest = request.addHeader("Authorization", "Bearer token")
        return proceed(authedRequest)  // 传给下一个
    }
}

class LogInterceptor : Interceptor() {
    override fun intercept(request: Request): Response {
        println("请求: ${request.url}")
        val response = proceed(request)
        println("响应: ${response.code}")
        return response
    }
}

class NetworkInterceptor : Interceptor() {
    override fun intercept(request: Request): Response {
        return Response(200, "OK")  // 实际发出请求
    }
}

// 构建链
fun buildChain(): Interceptor {
    val auth = AuthInterceptor()
    val log = LogInterceptor()
    val network = NetworkInterceptor()
    auth.next = log
    log.next = network
    return auth
}
```

---

### 9. 模板方法模式（Template Method）

#### 代码示例

```java
// Activity 生命周期就是模板方法模式
public abstract class BaseActivity extends AppCompatActivity {

    @Override
    protected final void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        initView();     // 抽象方法，子类实现
        initData();     // 抽象方法，子类实现
        setupListener();// 可选，子类重写
    }

    protected abstract int getLayoutId();
    protected abstract void initView();
    protected abstract void initData();

    // 钩子方法（有默认实现，子类可选择重写）
    protected void setupListener() {}
}

public class UserActivity extends BaseActivity {
    @Override
    protected int getLayoutId() { return R.layout.activity_user; }

    @Override
    protected void initView() {
        // 初始化 View
    }

    @Override
    protected void initData() {
        // 加载数据
    }
}
```

---

### 10. MVP 中如何防止 Presenter 内存泄漏

#### 代码示例

```java
// 问题：Presenter 持有 View（Activity）引用
// 当 Activity 销毁后，Presenter 还在，导致 Activity 无法被 GC

// 解决方案 1：弱引用
public class LoginPresenter {
    private WeakReference<LoginView> viewRef;

    public void attachView(LoginView view) {
        viewRef = new WeakReference<>(view);
    }

    public void detachView() {
        if (viewRef != null) {
            viewRef.clear();
            viewRef = null;
        }
    }

    private void onLoginSuccess(User user) {
        LoginView view = viewRef != null ? viewRef.get() : null;
        if (view != null) {
            view.showLoginSuccess(user);
        }
    }
}

// Activity 中
@Override
protected void onDestroy() {
    super.onDestroy();
    presenter.detachView();  // 断开引用
}

// 解决方案 2：基类统一处理
public abstract class BasePresenter<V> {
    private WeakReference<V> viewRef;

    public void attachView(V view) { viewRef = new WeakReference<>(view); }
    public void detachView() { if (viewRef != null) { viewRef.clear(); viewRef = null; } }
    protected V getView() { return viewRef != null ? viewRef.get() : null; }
    protected boolean isViewAttached() { return viewRef != null && viewRef.get() != null; }
}
```

---

### 11. 常用设计模式在 Android 中的应用

#### 详细答案

```
Android 中设计模式的体现：

创建型：
├─ 单例：Application、SystemServiceManager
├─ 工厂：BitmapFactory、LayoutInflater
└─ 建造者：AlertDialog.Builder、Notification.Builder

结构型：
├─ 装饰者：ContextWrapper（包装 Context）
├─ 代理：Retrofit（动态代理）、Binder（远程代理）
├─ 适配器：RecyclerView.Adapter
└─ 外观：Glide（简单接口包装复杂实现）

行为型：
├─ 观察者：LiveData、RxJava、BroadcastReceiver
├─ 责任链：OkHttp 拦截器链、事件分发
├─ 策略：LayoutManager（线性/网格/瀑布流）
├─ 模板方法：Activity/Fragment 生命周期
└─ 命令：Handler.post(Runnable)
```

---

## 📊 Part 4 完成总结

### ✅ 已完成（40 题）

**Kotlin（11 题）：**
1. ✅ Kotlin 基本特性（空安全、数据类、扩展函数）
2. ✅ 协程原理和使用（launch/async/Flow/StateFlow）
3. ✅ 高阶函数和 Lambda（map/filter/作用域函数）
4. ✅ 密封类（Sealed Class）
5. ✅ 委托属性（lazy/observable/自定义）
6. ✅ 泛型和型变（out/in/reified）
7. ✅ data class vs 普通类
8. ✅ 协程 vs 线程对比
9. ✅ object 关键字三种用法
10. ✅ inline/noinline/crossinline
11. ✅ Kotlin 与 Java 互操作

**数据结构（7 题）：**
12. ✅ 数组 vs 链表（手写链表、快慢指针）
13. ✅ HashMap 实现原理（数组+链表+红黑树）
14. ✅ 二叉树和 BST（前中后序、层序遍历）
15. ✅ 排序算法对比（快排、归并排序）
16. ✅ 栈和队列（双栈实现队列、单调栈）
17. ✅ 红黑树原理
18. ✅ LruCache 原理（手写实现）

**设计模式（11 题）：**
19. ✅ 单例模式（DCL/静态内部类/枚举/Kotlin object）
20. ✅ 观察者模式（LiveData/Flow/EventBus）
21. ✅ 工厂模式（简单工厂/抽象工厂）
22. ✅ 建造者模式
23. ✅ 策略模式
24. ✅ 装饰者模式（IO 流 / ContextWrapper）
25. ✅ 代理模式（静态代理/动态代理/Retrofit）
26. ✅ 责任链模式（OkHttp 拦截器）
27. ✅ 模板方法模式（Activity 生命周期）
28. ✅ MVP 防内存泄漏
29. ✅ 设计模式在 Android 中的应用汇总

---

## 📚 四部分完整汇总

| Part | 主题 | 题数 |
|------|------|------|
| **Part 1** | 四大组件 + 异步消息机制 | 22 题 |
| **Part 2** | Android UI 绘制 + 性能优化 | 45 题 |
| **Part 3** | 架构设计 + 网络 + 存储 + IPC | 35 题 |
| **Part 4** | Kotlin + 数据结构 + 设计模式 | 40 题 |
| **🎉 合计** | | **142 题** |

---

**完整 Android 面试题库已覆盖 142 道核心题目！** 🎉

建议学习路径：
```
1. Part 1 → 打基础（四大组件、Handler）
2. Part 2 → 攻UI（绘制机制、性能优化）
3. Part 3 → 懂架构（Jetpack、网络、IPC）
4. Part 4 → 提层次（Kotlin、算法、设计模式）
```