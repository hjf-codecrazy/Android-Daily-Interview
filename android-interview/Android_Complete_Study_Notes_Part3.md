# Android 完整学习笔记 - Part 3

> 包含详细答案 + 代码示例，适合深度学习和理解
>
> 共 35 题：架构设计（10题）+ 网络通信（8题）+ 数据存储（7题）+ 进程间通信（10题）

---

## 📌 架构设计

### 1. MVC、MVP、MVVM 的区别和选择

#### 详细答案

| 架构 | 全称 | 核心思想 | 适用场景 |
|------|------|---------|---------|
| **MVC** | Model-View-Controller | View 直接持有 Model | 简单项目，快速开发 |
| **MVP** | Model-View-Presenter | View 和 Model 完全解耦 | 中型项目，测试友好 |
| **MVVM** | Model-View-ViewModel | 双向数据绑定 | 大型项目，Jetpack推荐 |

**数据流向：**
```
MVC：
View ←→ Controller ←→ Model
View 直接读 Model（耦合高）

MVP：
View ↔ Presenter ↔ Model
View 只通过接口与 Presenter 通信

MVVM：
View ← ViewModel ← Model
View 观察 ViewModel，ViewModel 不知道 View
```

#### 代码示例

```java
// ========== MVP 示例 ==========

// 1. 契约接口（定义 View 和 Presenter 的方法）
public interface LoginContract {

    interface View {
        void showLoading();
        void hideLoading();
        void showLoginSuccess(String username);
        void showLoginError(String error);
    }

    interface Presenter {
        void login(String username, String password);
        void destroy();
    }
}

// 2. Model（数据层）
public class LoginModel {
    public void login(String username, String password,
                      LoginCallback callback) {
        // 模拟网络请求
        new Thread(() -> {
            try {
                Thread.sleep(1000);
                if ("admin".equals(username) && "123456".equals(password)) {
                    callback.onSuccess(username);
                } else {
                    callback.onError("用户名或密码错误");
                }
            } catch (InterruptedException e) {
                callback.onError(e.getMessage());
            }
        }).start();
    }

    public interface LoginCallback {
        void onSuccess(String username);
        void onError(String error);
    }
}

// 3. Presenter（业务逻辑层）
public class LoginPresenter implements LoginContract.Presenter {
    private final LoginContract.View view;
    private final LoginModel model;

    public LoginPresenter(LoginContract.View view) {
        this.view = view;
        this.model = new LoginModel();
    }

    @Override
    public void login(String username, String password) {
        if (username.isEmpty() || password.isEmpty()) {
            view.showLoginError("用户名和密码不能为空");
            return;
        }

        view.showLoading();
        model.login(username, password, new LoginModel.LoginCallback() {
            @Override
            public void onSuccess(String name) {
                // 切换到主线程
                new Handler(Looper.getMainLooper()).post(() -> {
                    view.hideLoading();
                    view.showLoginSuccess(name);
                });
            }

            @Override
            public void onError(String error) {
                new Handler(Looper.getMainLooper()).post(() -> {
                    view.hideLoading();
                    view.showLoginError(error);
                });
            }
        });
    }

    @Override
    public void destroy() {
        // 取消网络请求等清理工作
    }
}

// 4. View（Activity 实现 View 接口）
public class LoginActivity extends AppCompatActivity implements LoginContract.View {
    private LoginPresenter presenter;
    private ProgressBar progressBar;
    private EditText etUsername, etPassword;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        presenter = new LoginPresenter(this);

        findViewById(R.id.btn_login).setOnClickListener(v -> {
            String username = etUsername.getText().toString();
            String password = etPassword.getText().toString();
            presenter.login(username, password);
        });
    }

    @Override
    public void showLoading() {
        progressBar.setVisibility(View.VISIBLE);
    }

    @Override
    public void hideLoading() {
        progressBar.setVisibility(View.GONE);
    }

    @Override
    public void showLoginSuccess(String username) {
        Toast.makeText(this, "欢迎 " + username, Toast.LENGTH_SHORT).show();
    }

    @Override
    public void showLoginError(String error) {
        Toast.makeText(this, error, Toast.LENGTH_SHORT).show();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        presenter.destroy();  // 防止内存泄漏
    }
}
```

```kotlin
// ========== MVVM 示例（Kotlin + Jetpack）==========

// 1. Model（数据层 + Repository）
class LoginRepository {
    suspend fun login(username: String, password: String): Result<String> {
        return withContext(Dispatchers.IO) {
            delay(1000)  // 模拟网络请求
            if (username == "admin" && password == "123456") {
                Result.success(username)
            } else {
                Result.failure(Exception("用户名或密码错误"))
            }
        }
    }
}

// 2. ViewModel（逻辑层，持有 LiveData）
class LoginViewModel : ViewModel() {
    private val repository = LoginRepository()

    // 使用 LiveData 暴露状态
    private val _loginState = MutableLiveData<LoginState>()
    val loginState: LiveData<LoginState> = _loginState

    fun login(username: String, password: String) {
        if (username.isEmpty() || password.isEmpty()) {
            _loginState.value = LoginState.Error("用户名和密码不能为空")
            return
        }

        _loginState.value = LoginState.Loading
        viewModelScope.launch {
            repository.login(username, password)
                .onSuccess { name ->
                    _loginState.value = LoginState.Success(name)
                }
                .onFailure { error ->
                    _loginState.value = LoginState.Error(error.message ?: "登录失败")
                }
        }
    }
}

// 3. 状态类（密封类）
sealed class LoginState {
    object Loading : LoginState()
    data class Success(val username: String) : LoginState()
    data class Error(val message: String) : LoginState()
}

// 4. View（Activity 观察 ViewModel）
class LoginActivity : AppCompatActivity() {
    private val viewModel: LoginViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        // 观察状态变化（生命周期感知）
        viewModel.loginState.observe(this) { state ->
            when (state) {
                is LoginState.Loading -> showLoading()
                is LoginState.Success -> showSuccess(state.username)
                is LoginState.Error -> showError(state.message)
            }
        }

        binding.btnLogin.setOnClickListener {
            viewModel.login(
                binding.etUsername.text.toString(),
                binding.etPassword.text.toString()
            )
        }
    }
}
```

#### 核心要点

```
架构对比：

MVC
├─ 优点：简单，开发速度快
└─ 缺点：View 和 Model 耦合，Activity 臃肿

MVP
├─ 优点：View 和 Model 解耦，可单元测试
├─ 缺点：Presenter 和 View 接口一一对应，代码量大
└─ 防泄漏：onDestroy 调用 presenter.destroy()

MVVM（推荐）
├─ 优点：ViewModel 生命周期感知，数据双向绑定
├─ 优点：配合 Jetpack（LiveData/Flow）更强大
└─ 缺点：调试相对复杂
```

---

### 2. Jetpack ViewModel 的原理

#### 详细答案

ViewModel 的核心作用：**在配置变更（如屏幕旋转）时保存数据**。

**内部原理：**
```
Activity 旋转时
    ↓
ViewModelStore 保存 ViewModel
    ↓
新 Activity 创建
    ↓
从 ViewModelStore 中取出同一个 ViewModel
    ↓
ViewModel 中的数据未丢失
```

ViewModel 存储在 `ViewModelStore` 中，而 `ViewModelStore` 在 Activity 重建时不会被销毁（通过 `onRetainNonConfigurationInstance()` 保存）。

#### 代码示例

```kotlin
// 1. 基本使用
class CounterViewModel : ViewModel() {
    var count = 0
        private set

    fun increment() {
        count++
    }

    // ViewModel 销毁时调用（真正销毁，非旋转）
    override fun onCleared() {
        super.onCleared()
        // 取消协程、释放资源
    }
}

class CounterActivity : AppCompatActivity() {
    // by viewModels() 委托 - 自动管理生命周期
    private val viewModel: CounterViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 旋转屏幕后，viewModel 是同一个实例
        binding.tvCount.text = viewModel.count.toString()
    }
}

// 2. Fragment 共享 ViewModel
class SharedViewModel : ViewModel() {
    val selectedItem = MutableLiveData<String>()
}

class ListFragment : Fragment() {
    // activityViewModels() - 与 Activity 共享同一个 ViewModel
    private val sharedViewModel: SharedViewModel by activityViewModels()

    fun onItemClick(item: String) {
        sharedViewModel.selectedItem.value = item
    }
}

class DetailFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        sharedViewModel.selectedItem.observe(viewLifecycleOwner) { item ->
            // 接收来自 ListFragment 的数据
        }
    }
}

// 3. ViewModel + SavedStateHandle（进程死亡也能恢复）
class SavedStateViewModel(private val savedState: SavedStateHandle) : ViewModel() {

    // 从 SavedState 读取（进程被杀死后仍可恢复）
    val username: LiveData<String> = savedState.getLiveData("username", "")

    fun setUsername(name: String) {
        savedState["username"] = name
    }
}
```

#### 核心要点

```
ViewModel 生命周期：
Activity 创建 → ViewModel 创建
Activity 旋转 → ViewModel 继续存在（数据不丢失）
Activity 真正销毁 → ViewModel.onCleared() → ViewModel 销毁

与 onSaveInstanceState 的区别：
├─ ViewModel：存在内存中，数据量不限，进程死亡则丢失
└─ onSaveInstanceState：持久化，限大小，可跨进程存活

创建方式：
├─ by viewModels()：Activity/Fragment 独享
├─ by activityViewModels()：与 Activity 共享
└─ ViewModelProvider(activity)[MyViewModel::class.java]：手动创建
```

---

### 3. LiveData 的工作原理

#### 详细答案

LiveData 是一个**可观察的数据持有者**，它具备**生命周期感知**能力——只在 Activity/Fragment 处于活跃状态时通知观察者。

**核心机制：**
- 内部维护观察者列表（Observer + LifecycleOwner）
- 当 LifecycleOwner 变为 DESTROYED 时自动移除观察者
- 只在 STARTED 或 RESUMED 状态时分发事件

#### 代码示例

```kotlin
// 1. 基本 LiveData
class UserViewModel : ViewModel() {
    // MutableLiveData 可以修改值
    private val _user = MutableLiveData<User>()
    // 对外暴露只读的 LiveData
    val user: LiveData<User> = _user

    fun loadUser(userId: String) {
        viewModelScope.launch {
            val result = repository.getUser(userId)
            _user.postValue(result)  // 后台线程用 postValue
            // _user.value = result   // 主线程用 value
        }
    }
}

// 2. 观察 LiveData
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 生命周期感知：Activity 不可见时不会收到通知
        viewModel.user.observe(this) { user ->
            binding.tvName.text = user.name
        }
    }
}

// 3. Transformations（LiveData 转换）
class TransformViewModel : ViewModel() {
    private val userId = MutableLiveData<String>()

    // map：转换 LiveData 的值
    val userName: LiveData<String> = userId.map { id ->
        "User: $id"
    }

    // switchMap：根据值切换到另一个 LiveData
    val userDetail: LiveData<User> = userId.switchMap { id ->
        repository.getUserLiveData(id)  // 返回新的 LiveData
    }

    fun changeUser(id: String) {
        userId.value = id
    }
}

// 4. MediatorLiveData（合并多个 LiveData）
class MergedViewModel : ViewModel() {
    val liveData1 = MutableLiveData<String>()
    val liveData2 = MutableLiveData<String>()

    val merged = MediatorLiveData<String>().apply {
        addSource(liveData1) { value = "来自1: $it" }
        addSource(liveData2) { value = "来自2: $it" }
    }
}

// 5. SingleLiveEvent（只消费一次的事件，如导航、Toast）
class SingleLiveEvent<T> : MutableLiveData<T>() {
    private val pending = AtomicBoolean(false)

    override fun observe(owner: LifecycleOwner, observer: Observer<in T>) {
        super.observe(owner) { t ->
            if (pending.compareAndSet(true, false)) {
                observer.onChanged(t)
            }
        }
    }

    override fun setValue(t: T?) {
        pending.set(true)
        super.setValue(t)
    }
}
```

#### 核心要点

```
LiveData 特点：
├─ 生命周期感知：不活跃时不推送数据
├─ 自动取消订阅：onDestroy 后自动移除观察者
├─ 数据粘性：新观察者会立即收到最新值（注意 SingleLiveEvent）
└─ 线程安全：setValue（主线程）/ postValue（子线程）

常用 API：
├─ observe()：生命周期感知观察
├─ observeForever()：永久观察（需手动 removeObserver）
├─ map()：变换数据类型
├─ switchMap()：切换数据源
└─ MediatorLiveData：合并多个 LiveData
```

---

### 4. Room 数据库的使用

#### 详细答案

Room 是 Jetpack 提供的 SQLite 抽象层，提供编译时 SQL 检查和简洁的 API。

**三个核心组件：**
- **Entity**：数据表（用 @Entity 注解的数据类）
- **DAO**：数据访问对象（定义 SQL 操作）
- **Database**：数据库入口

#### 代码示例

```kotlin
// 1. Entity（数据表）
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,

    @ColumnInfo(name = "user_name")
    val name: String,

    val email: String,

    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis()
)

// 2. DAO（数据访问对象）
@Dao
interface UserDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User): Long

    @Insert
    suspend fun insertUsers(users: List<User>)

    @Update
    suspend fun updateUser(user: User)

    @Delete
    suspend fun deleteUser(user: User)

    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>  // Flow 自动更新

    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: Long): User?

    @Query("SELECT * FROM users WHERE user_name LIKE :name")
    suspend fun searchUsers(name: String): List<User>

    @Query("DELETE FROM users WHERE id = :userId")
    suspend fun deleteById(userId: Long)

    @Transaction  // 事务操作
    suspend fun insertAndDelete(newUser: User, oldUser: User) {
        insertUser(newUser)
        deleteUser(oldUser)
    }
}

// 3. Database（数据库）
@Database(
    entities = [User::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2)  // 数据库迁移
                .build()
                INSTANCE = instance
                instance
            }
        }

        // 数据库升级迁移
        private val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("ALTER TABLE users ADD COLUMN phone TEXT")
            }
        }
    }
}

// 4. Repository（数据仓库）
class UserRepository(private val userDao: UserDao) {

    val allUsers: Flow<List<User>> = userDao.getAllUsers()

    suspend fun insertUser(user: User) = userDao.insertUser(user)

    suspend fun getUserById(id: Long) = userDao.getUserById(id)
}

// 5. ViewModel 中使用
class UserViewModel(application: Application) : AndroidViewModel(application) {
    private val repository: UserRepository

    init {
        val db = AppDatabase.getInstance(application)
        repository = UserRepository(db.userDao())
    }

    val users: LiveData<List<User>> = repository.allUsers.asLiveData()

    fun addUser(name: String, email: String) {
        viewModelScope.launch {
            repository.insertUser(User(name = name, email = email))
        }
    }
}
```

#### 核心要点

```
Room 三层结构：
Entity → 数据表（@Entity）
DAO   → 增删改查接口（@Dao）
DB    → 单例入口（@Database）

优点：
├─ 编译时 SQL 语法检查
├─ 支持 Flow/LiveData 响应式查询
├─ 协程支持（suspend 函数）
└─ 类型安全

数据库升级：
├─ 修改 version
├─ 添加 Migration 对象
└─ 或 fallbackToDestructiveMigration()（开发时用）
```

---

### 5. Jetpack Navigation 的使用

#### 详细答案

Navigation 组件管理 Fragment 导航，提供可视化导航图、安全参数传递和深链接支持。

#### 代码示例

```xml
<!-- res/navigation/nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment">
        <argument
            android:name="itemId"
            app:argType="string" />
    </fragment>
</navigation>
```

```kotlin
// HomeFragment 导航到 DetailFragment
class HomeFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.btnDetail.setOnClickListener {
            // 使用 Safe Args 传参（需要 navigation-safe-args 插件）
            val action = HomeFragmentDirections
                .actionHomeToDetail(itemId = "123")
            findNavController().navigate(action)

            // 或者直接用 Bundle
            val bundle = bundleOf("itemId" to "123")
            findNavController().navigate(R.id.action_home_to_detail, bundle)
        }
    }
}

// DetailFragment 接收参数
class DetailFragment : Fragment() {
    // Safe Args 自动生成的参数类
    private val args: DetailFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val itemId = args.itemId
        binding.tvId.text = itemId

        // 返回上一页
        binding.btnBack.setOnClickListener {
            findNavController().popBackStack()
        }
    }
}

// Activity 中设置 NavController
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val navController = findNavController(R.id.nav_host_fragment)
        // 设置 ActionBar 与 Navigation 联动
        setupActionBarWithNavController(navController)
    }

    override fun onSupportNavigateUp(): Boolean {
        return findNavController(R.id.nav_host_fragment).navigateUp()
                || super.onSupportNavigateUp()
    }
}
```

#### 核心要点

```
Navigation 优势：
├─ 可视化导航图，清晰管理页面关系
├─ Safe Args：编译时类型安全的参数传递
├─ 深链接（Deep Link）支持
└─ 统一处理回退栈

常用 API：
├─ findNavController().navigate()：导航到目标
├─ findNavController().popBackStack()：返回
├─ findNavController().navigateUp()：向上导航
└─ navArgs()：接收 Safe Args 参数
```

---

### 6. DataBinding 的使用和原理

#### 详细答案

DataBinding 让布局文件直接绑定数据，减少 `findViewById` 和样板代码。

#### 代码示例

```xml
<!-- activity_user.xml - 使用 DataBinding -->
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable
            name="user"
            type="com.example.User" />
        <variable
            name="viewModel"
            type="com.example.UserViewModel" />
    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <!-- 单向绑定 -->
        <TextView
            android:text="@{user.name}"
            android:visibility="@{user.isVip ? View.VISIBLE : View.GONE}" />

        <!-- 双向绑定（EditText） -->
        <EditText
            android:text="@={viewModel.inputName}" />

        <!-- 事件绑定 -->
        <Button
            android:onClick="@{() -> viewModel.onLoginClick()}" />

    </LinearLayout>
</layout>
```

```kotlin
// Activity 中使用 DataBinding
class UserActivity : AppCompatActivity() {
    private lateinit var binding: ActivityUserBinding
    private val viewModel: UserViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 替代 setContentView
        binding = DataBindingUtil.setContentView(this, R.layout.activity_user)

        // 绑定数据
        binding.user = User("张三", true)
        binding.viewModel = viewModel

        // 设置生命周期所有者（LiveData 自动更新 UI）
        binding.lifecycleOwner = this
    }
}

// ViewModel 中双向绑定
class UserViewModel : ViewModel() {
    // 双向绑定需要 MutableLiveData
    val inputName = MutableLiveData<String>("")

    fun onLoginClick() {
        val name = inputName.value
        Log.d("Login", "输入的名字: $name")
    }
}

// 自定义 BindingAdapter
object BindingAdapters {
    @JvmStatic
    @BindingAdapter("imageUrl")
    fun loadImage(imageView: ImageView, url: String?) {
        Glide.with(imageView.context)
            .load(url)
            .into(imageView)
    }

    @JvmStatic
    @BindingAdapter("app:visibleOrGone")
    fun setVisibleOrGone(view: View, visible: Boolean) {
        view.visibility = if (visible) View.VISIBLE else View.GONE
    }
}
```

```xml
<!-- 使用自定义 BindingAdapter -->
<ImageView
    app:imageUrl="@{user.avatarUrl}" />
```

#### 核心要点

```
DataBinding 原理：
├─ 编译时生成 *Binding 类（如 ActivityMainBinding）
├─ @{} 单向绑定：数据 → UI
├─ @={} 双向绑定：数据 ↔ UI（需要 MutableLiveData）
└─ 通过 binding.lifecycleOwner 实现 LiveData 自动更新

自定义 BindingAdapter：
├─ @BindingAdapter("属性名") 注解静态方法
└─ 可以自定义任何 XML 属性的处理逻辑

优点：
├─ 消除 findViewById
├─ 减少代码量
└─ 编译时检查（属性拼写错误会报编译错误）
```

---

### 7. WorkManager 的使用场景

#### 详细答案

WorkManager 用于执行**可延迟的、保证执行**的后台任务，即使 App 关闭或设备重启也能保证任务执行。

#### 代码示例

```kotlin
// 1. 定义 Worker
class SyncDataWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            // 执行后台任务（如同步数据）
            val userId = inputData.getString("userId") ?: ""
            syncData(userId)

            // 输出结果
            val outputData = workDataOf("result" to "同步成功")
            Result.success(outputData)
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()  // 重试
            } else {
                Result.failure()  // 失败
            }
        }
    }

    private suspend fun syncData(userId: String) {
        // 执行实际的同步逻辑
        delay(2000)
    }
}

// 2. 提交任务
class WorkManagerDemo {

    fun scheduleOneTimeWork(context: Context) {
        // 构建约束条件
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)  // 需要网络
            .setRequiresBatteryNotLow(true)  // 电量不低
            .build()

        // 构建一次性任务
        val inputData = workDataOf("userId" to "123")
        val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
            .setConstraints(constraints)
            .setInputData(inputData)
            .setBackoffCriteria(
                BackoffPolicy.EXPONENTIAL,
                WorkRequest.MIN_BACKOFF_MILLIS,
                TimeUnit.MILLISECONDS
            )
            .build()

        WorkManager.getInstance(context)
            .enqueueUniqueWork(
                "sync_data",
                ExistingWorkPolicy.KEEP,  // 已有任务则保留
                workRequest
            )
    }

    fun schedulePeriodicWork(context: Context) {
        // 周期性任务（最小间隔 15 分钟）
        val periodicWork = PeriodicWorkRequestBuilder<SyncDataWorker>(
            15, TimeUnit.MINUTES
        ).build()

        WorkManager.getInstance(context)
            .enqueueUniquePeriodicWork(
                "periodic_sync",
                ExistingPeriodicWorkPolicy.KEEP,
                periodicWork
            )
    }

    fun observeWork(context: Context, workId: UUID) {
        WorkManager.getInstance(context)
            .getWorkInfoByIdLiveData(workId)
            .observe(lifecycleOwner) { workInfo ->
                when (workInfo.state) {
                    WorkInfo.State.RUNNING -> showProgress()
                    WorkInfo.State.SUCCEEDED -> {
                        val result = workInfo.outputData.getString("result")
                        showSuccess(result)
                    }
                    WorkInfo.State.FAILED -> showError()
                    else -> {}
                }
            }
    }

    // 任务链
    fun chainWork(context: Context) {
        val task1 = OneTimeWorkRequestBuilder<SyncDataWorker>().build()
        val task2 = OneTimeWorkRequestBuilder<SyncDataWorker>().build()
        val task3 = OneTimeWorkRequestBuilder<SyncDataWorker>().build()

        WorkManager.getInstance(context)
            .beginWith(task1)
            .then(task2)
            .then(task3)
            .enqueue()
    }
}
```

#### 核心要点

```
WorkManager 适用场景：
├─ 数据同步、上传日志
├─ 图片压缩、文件处理
├─ 定时备份
└─ 任何需要保证执行的后台任务

三种 Result：
├─ Result.success()：成功
├─ Result.failure()：失败（不再重试）
└─ Result.retry()：稍后重试

与其他方案对比：
├─ Service：不保证执行，系统可能杀死
├─ AlarmManager：需要自己处理重试
├─ JobScheduler：API 21+，WorkManager 内部使用它
└─ WorkManager：最推荐，API 14+ 兼容
```

---

### 8. Paging 分页库的使用

#### 详细答案

Paging 3 提供了完整的分页解决方案，配合 RecyclerView 实现无感知加载更多。

#### 代码示例

```kotlin
// 1. 定义 PagingSource
class UserPagingSource(
    private val api: UserApi,
    private val query: String
) : PagingSource<Int, User>() {

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        val page = params.key ?: 1  // 初始页码
        return try {
            val response = api.getUsers(query = query, page = page, pageSize = params.loadSize)
            LoadResult.Page(
                data = response.users,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.users.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<Int, User>): Int? {
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }
}

// 2. Repository 层
class UserRepository(private val api: UserApi) {
    fun searchUsers(query: String): Flow<PagingData<User>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                enablePlaceholders = false,
                prefetchDistance = 3  // 距底部 3 个时预加载
            ),
            pagingSourceFactory = { UserPagingSource(api, query) }
        ).flow
    }
}

// 3. ViewModel
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    private val query = MutableStateFlow("")

    val users: Flow<PagingData<User>> = query
        .debounce(300)  // 防抖
        .flatMapLatest { repository.searchUsers(it) }
        .cachedIn(viewModelScope)  // 缓存到 ViewModel 范围

    fun search(newQuery: String) {
        query.value = newQuery
    }
}

// 4. Adapter
class UserPagingAdapter : PagingDataAdapter<User, UserViewHolder>(USER_COMPARATOR) {

    companion object {
        val USER_COMPARATOR = object : DiffUtil.ItemCallback<User>() {
            override fun areItemsTheSame(old: User, new: User) = old.id == new.id
            override fun areContentsTheSame(old: User, new: User) = old == new
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        val binding = ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return UserViewHolder(binding)
    }

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        getItem(position)?.let { holder.bind(it) }
    }
}

// 5. Fragment 中使用
class UserFragment : Fragment() {
    private val viewModel: UserViewModel by viewModels()
    private val adapter = UserPagingAdapter()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.recyclerView.adapter = adapter.withLoadStateFooter(
            footer = LoadStateAdapter { adapter.retry() }  // 加载更多的 Footer
        )

        lifecycleScope.launch {
            viewModel.users.collectLatest { pagingData ->
                adapter.submitData(pagingData)
            }
        }

        // 监听加载状态
        adapter.addLoadStateListener { loadState ->
            binding.progressBar.isVisible = loadState.refresh is LoadState.Loading
            binding.retryButton.isVisible = loadState.refresh is LoadState.Error
        }
    }
}
```

#### 核心要点

```
Paging 3 核心组件：
├─ PagingSource：定义数据来源（网络/本地）
├─ PagingData：分页数据容器（Flow<PagingData>）
├─ PagingDataAdapter：RecyclerView Adapter
└─ cachedIn(viewModelScope)：防止重复加载

LoadState 三种状态：
├─ Loading：加载中
├─ NotLoading：未在加载
└─ Error：加载失败（可调用 retry()）
```

---

### 9. Glide 图片加载库原理

#### 详细答案

Glide 是一个高效的图片加载库，核心在于**三级缓存**和**生命周期绑定**。

**缓存机制（三级缓存）：**
```
内存缓存（LruCache）→ 磁盘缓存（DiskLruCache）→ 网络请求
```

#### 代码示例

```java
// 1. 基本使用
Glide.with(context)
    .load(url)
    .placeholder(R.drawable.placeholder)  // 占位图
    .error(R.drawable.error)              // 错误图
    .centerCrop()                         // 裁剪方式
    .into(imageView);

// 2. 高级配置
RequestOptions options = new RequestOptions()
    .diskCacheStrategy(DiskCacheStrategy.ALL)  // 磁盘缓存策略
    .override(200, 200)                        // 固定尺寸
    .priority(Priority.HIGH)                   // 优先级
    .skipMemoryCache(false);                   // 使用内存缓存

Glide.with(context)
    .load(url)
    .apply(options)
    .into(imageView);

// 3. 生命周期绑定（自动取消）
// with(Activity) - 随 Activity 生命周期
// with(Fragment) - 随 Fragment 生命周期
// with(ApplicationContext) - 全局，不自动取消

// 4. 监听加载结果
Glide.with(context)
    .load(url)
    .listener(new RequestListener<Drawable>() {
        @Override
        public boolean onLoadFailed(GlideException e, Object model,
                                     Target<Drawable> target, boolean isFirstResource) {
            Log.e("Glide", "加载失败", e);
            return false;  // 返回 true 则不显示 error 图
        }

        @Override
        public boolean onResourceReady(Drawable resource, Object model,
                                        Target<Drawable> target,
                                        DataSource dataSource, boolean isFirstResource) {
            Log.d("Glide", "加载成功，来源: " + dataSource);
            return false;  // 返回 true 则不自动设置图片
        }
    })
    .into(imageView);

// 5. 清除缓存
// 清除内存缓存（主线程）
Glide.get(context).clearMemory();

// 清除磁盘缓存（后台线程）
new Thread(() -> Glide.get(context).clearDiskCache()).start();
```

#### 核心要点

```
Glide 三级缓存：
1. 活跃内存缓存（WeakReference）：正在使用的图片
2. 内存缓存（LruCache）：最近使用的图片
3. 磁盘缓存（DiskLruCache）：本地文件

DiskCacheStrategy：
├─ ALL：缓存原图和变换后的图
├─ NONE：不缓存
├─ DATA：只缓存原图
├─ RESOURCE：只缓存变换后的图
└─ AUTOMATIC（默认）：智能选择

生命周期绑定原理：
└─ Glide 向 Activity/Fragment 注入一个空白 Fragment
   监听其生命周期来暂停/继续/取消图片加载
```

---

### 10. 什么是 MVVM + Clean Architecture？

#### 详细答案

Clean Architecture 将代码分为三层：**表示层 → 领域层 → 数据层**，配合 MVVM 实现关注点分离。

```
表示层（Presentation）
├─ Activity / Fragment（View）
├─ ViewModel
└─ UI State

领域层（Domain）
├─ UseCase（业务用例，如 LoginUseCase）
├─ Repository 接口
└─ Entity（业务模型）

数据层（Data）
├─ Repository 实现
├─ Remote（API 接口）
└─ Local（Room / SP）
```

#### 代码示例

```kotlin
// 领域层 - UseCase
class LoginUseCase(private val authRepository: AuthRepository) {
    suspend operator fun invoke(username: String, password: String): Result<User> {
        if (username.isEmpty()) return Result.failure(IllegalArgumentException("用户名不能为空"))
        return authRepository.login(username, password)
    }
}

// 领域层 - Repository 接口
interface AuthRepository {
    suspend fun login(username: String, password: String): Result<User>
    suspend fun logout()
    fun getCurrentUser(): Flow<User?>
}

// 数据层 - Repository 实现
class AuthRepositoryImpl(
    private val remoteDataSource: AuthRemoteDataSource,
    private val localDataSource: AuthLocalDataSource
) : AuthRepository {

    override suspend fun login(username: String, password: String): Result<User> {
        return try {
            val userDto = remoteDataSource.login(username, password)
            val user = userDto.toDomain()  // DTO -> 领域模型转换
            localDataSource.saveUser(user)
            Result.success(user)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override fun getCurrentUser(): Flow<User?> = localDataSource.getUserFlow()
}

// 表示层 - ViewModel
class LoginViewModel(private val loginUseCase: LoginUseCase) : ViewModel() {
    private val _state = MutableStateFlow<LoginUiState>(LoginUiState.Idle)
    val state: StateFlow<LoginUiState> = _state

    fun login(username: String, password: String) {
        viewModelScope.launch {
            _state.value = LoginUiState.Loading
            loginUseCase(username, password)
                .onSuccess { user -> _state.value = LoginUiState.Success(user) }
                .onFailure { e -> _state.value = LoginUiState.Error(e.message ?: "") }
        }
    }
}

sealed class LoginUiState {
    object Idle : LoginUiState()
    object Loading : LoginUiState()
    data class Success(val user: User) : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```

#### 核心要点

```
Clean Architecture 三层：
├─ 表示层（Presentation）：UI + ViewModel
├─ 领域层（Domain）：UseCase（纯业务逻辑，无 Android 依赖）
└─ 数据层（Data）：网络 + 本地存储

依赖规则：外层依赖内层，内层不知道外层
表示层 → 领域层 ← 数据层

UseCase 好处：
├─ 单一职责：每个 UseCase 只做一件事
├─ 可复用：多个 ViewModel 复用同一 UseCase
└─ 可测试：不依赖 Android 框架，单元测试简单
```

---

## 📌 网络通信

### 1. HTTP 和 HTTPS 的区别

#### 详细答案

| 特性 | HTTP | HTTPS |
|------|------|-------|
| **端口** | 80 | 443 |
| **安全性** | 明文传输 | SSL/TLS 加密 |
| **证书** | 不需要 | 需要 CA 证书 |
| **速度** | 快 | 稍慢（握手开销） |
| **SEO** | 一般 | 更好 |

**HTTPS 握手流程：**
```
客户端 → 服务端：ClientHello（支持的加密算法）
服务端 → 客户端：ServerHello + 证书
客户端：验证证书（CA 签名）
客户端 → 服务端：用服务端公钥加密的随机数
双方：用三个随机数生成会话密钥（对称密钥）
后续：用会话密钥对称加密通信
```

#### 代码示例

```java
// OkHttp 配置 HTTPS（信任所有证书，仅用于测试）
public OkHttpClient createUnsafeClient() {
    try {
        TrustManager[] trustAllCerts = new TrustManager[]{
            new X509TrustManager() {
                @Override
                public void checkClientTrusted(X509Certificate[] chain, String authType) {}

                @Override
                public void checkServerTrusted(X509Certificate[] chain, String authType) {}

                @Override
                public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[]{}; }
            }
        };

        SSLContext sslContext = SSLContext.getInstance("SSL");
        sslContext.init(null, trustAllCerts, new SecureRandom());

        return new OkHttpClient.Builder()
            .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustAllCerts[0])
            .hostnameVerifier((hostname, session) -> true)
            .build();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}

// 证书固定（防止中间人攻击）
OkHttpClient client = new OkHttpClient.Builder()
    .certificatePinner(new CertificatePinner.Builder()
        .add("example.com", "sha256/AAAAAAAAAAAAAAAA")
        .build())
    .build();
```

---

### 2. OkHttp 原理和拦截器

#### 详细答案

OkHttp 通过**责任链模式**处理网络请求，核心是拦截器链。

**拦截器执行顺序：**
```
自定义拦截器（Application Interceptors）
    ↓
RetryAndFollowUpInterceptor（重试和重定向）
    ↓
BridgeInterceptor（请求/响应头处理）
    ↓
CacheInterceptor（缓存处理）
    ↓
ConnectInterceptor（建立连接）
    ↓
NetworkInterceptors（网络拦截器）
    ↓
CallServerInterceptor（实际发送请求）
```

#### 代码示例

```kotlin
// 1. 自定义拦截器（添加统一请求头）
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val newRequest = originalRequest.newBuilder()
            .header("Authorization", "Bearer ${tokenProvider.getToken()}")
            .header("Accept", "application/json")
            .build()
        return chain.proceed(newRequest)
    }
}

// 2. 日志拦截器
class LoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val t1 = System.nanoTime()
        Log.d("OkHttp", "请求 ${request.url}")

        val response = chain.proceed(request)
        val t2 = System.nanoTime()

        Log.d("OkHttp", "响应 ${response.code} 耗时 ${(t2 - t1) / 1e6}ms")
        return response
    }
}

// 3. 重试拦截器
class RetryInterceptor(private val maxRetry: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var retryCount = 0
        var response: Response
        do {
            try {
                response = chain.proceed(chain.request())
                if (response.isSuccessful) return response
            } catch (e: IOException) {
                if (retryCount >= maxRetry) throw e
            }
            retryCount++
        } while (retryCount <= maxRetry)
        return chain.proceed(chain.request())
    }
}

// 4. 构建 OkHttpClient
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenProvider))   // 应用拦截器（总是执行）
    .addInterceptor(LoggingInterceptor())
    .addNetworkInterceptor(RetryInterceptor())         // 网络拦截器（重定向时多次执行）
    .connectTimeout(15, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .cache(Cache(cacheDir, 10 * 1024 * 1024L))        // 10MB 缓存
    .build()
```

#### 核心要点

```
应用拦截器 vs 网络拦截器：
├─ addInterceptor：总执行一次，不管重定向
└─ addNetworkInterceptor：每次网络请求都执行（重定向会多次）

拦截器常见用途：
├─ 添加统一请求头（Token）
├─ 日志记录
├─ 请求重试
├─ 响应缓存
└─ 数据解密/加密
```

---

### 3. Retrofit 的原理

#### 详细答案

Retrofit 通过**动态代理**和**注解**将接口方法转换为 HTTP 请求。

#### 代码示例

```kotlin
// 1. 定义 API 接口
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") userId: String): UserResponse

    @GET("users")
    suspend fun getUsers(
        @Query("page") page: Int,
        @Query("size") size: Int
    ): PagedResponse<UserResponse>

    @POST("users/login")
    suspend fun login(@Body request: LoginRequest): LoginResponse

    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") userId: String,
        @Body user: UserRequest
    ): UserResponse

    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") userId: String): Response<Unit>

    @Multipart
    @POST("users/avatar")
    suspend fun uploadAvatar(
        @Part("userId") userId: RequestBody,
        @Part file: MultipartBody.Part
    ): AvatarResponse

    @Streaming  // 大文件下载
    @GET
    suspend fun downloadFile(@Url url: String): ResponseBody
}

// 2. 构建 Retrofit 实例
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .client(okHttpClient)
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val api = retrofit.create(UserApi::class.java)

// 3. Repository 中使用
class UserRepository(private val api: UserApi) {

    suspend fun getUser(userId: String): Result<User> {
        return try {
            val response = api.getUser(userId)
            Result.success(response.toDomain())
        } catch (e: HttpException) {
            // HTTP 错误（4xx, 5xx）
            Result.failure(Exception("HTTP ${e.code()}: ${e.message()}"))
        } catch (e: IOException) {
            // 网络错误
            Result.failure(Exception("网络连接失败"))
        }
    }

    suspend fun uploadAvatar(userId: String, file: File): Result<String> {
        val userIdBody = userId.toRequestBody("text/plain".toMediaTypeOrNull())
        val filePart = MultipartBody.Part.createFormData(
            "file",
            file.name,
            file.asRequestBody("image/*".toMediaTypeOrNull())
        )
        return try {
            val response = api.uploadAvatar(userIdBody, filePart)
            Result.success(response.avatarUrl)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

#### 核心要点

```
Retrofit 工作原理：
1. retrofit.create(UserApi::class.java)
   → 通过 Java 动态代理生成接口实现
2. 调用接口方法时，代理拦截调用
3. 解析注解（@GET/@POST/@Path/@Query）
4. 构建 OkHttp Request
5. 通过 OkHttp 发送请求
6. 通过 Converter 解析响应（Gson/Moshi）

常用注解：
├─ @GET/@POST/@PUT/@DELETE：HTTP 方法
├─ @Path：URL 路径参数
├─ @Query/@QueryMap：URL 查询参数
├─ @Body：请求体
├─ @Header/@Headers：请求头
└─ @Multipart/@Part：文件上传
```

---

### 4. TCP/IP 三次握手、四次挥手

#### 详细答案

**三次握手（建立连接）：**
```
客户端 → 服务端：SYN（seq=x）          → "我要连接"
服务端 → 客户端：SYN+ACK（seq=y, ack=x+1）→ "好的，准备好了"
客户端 → 服务端：ACK（ack=y+1）        → "收到，开始通信"
```

**四次挥手（断开连接）：**
```
客户端 → 服务端：FIN              → "我发完了"
服务端 → 客户端：ACK              → "好的，我知道了"
服务端 → 客户端：FIN              → "我也发完了"
客户端 → 服务端：ACK              → "好的，再见"
（客户端等待 2MSL 后真正关闭）
```

**为什么三次握手不是两次？**
防止历史连接（已失效的请求报文）被服务端误认为新连接。

**为什么四次挥手不是三次？**
TCP 是全双工的，服务端在收到 FIN 后，可能还有数据要发，所以需要分两步关闭。

#### 代码示例

```java
// Android 中 Socket 编程
public class SocketDemo {

    // TCP 客户端
    public void tcpClient() {
        new Thread(() -> {
            try (Socket socket = new Socket("192.168.1.1", 8080)) {
                // 发送数据
                PrintWriter writer = new PrintWriter(socket.getOutputStream(), true);
                writer.println("Hello Server");

                // 接收数据
                BufferedReader reader = new BufferedReader(
                    new InputStreamReader(socket.getInputStream()));
                String response = reader.readLine();
                Log.d("Socket", "收到: " + response);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }

    // TCP 服务端
    public void tcpServer() {
        new Thread(() -> {
            try (ServerSocket serverSocket = new ServerSocket(8080)) {
                while (true) {
                    Socket client = serverSocket.accept();  // 阻塞等待连接
                    handleClient(client);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }

    private void handleClient(Socket client) {
        new Thread(() -> {
            try {
                BufferedReader reader = new BufferedReader(
                    new InputStreamReader(client.getInputStream()));
                PrintWriter writer = new PrintWriter(client.getOutputStream(), true);

                String line;
                while ((line = reader.readLine()) != null) {
                    Log.d("Server", "收到: " + line);
                    writer.println("Echo: " + line);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

---

### 5. Android 网络请求优化

#### 详细答案

#### 代码示例

```kotlin
// 1. 请求合并（批量请求）
class BatchNetworkDemo {
    suspend fun loadPageData(): PageData {
        return coroutineScope {
            // 并行执行多个请求
            val userDeferred = async { api.getUser() }
            val newsDeferred = async { api.getNewsList() }
            val bannerDeferred = async { api.getBanners() }

            // 等待全部完成
            PageData(
                user = userDeferred.await(),
                news = newsDeferred.await(),
                banners = bannerDeferred.await()
            )
        }
    }
}

// 2. 请求缓存
val cacheInterceptor = Interceptor { chain ->
    var request = chain.request()
    val response = chain.proceed(request)

    // 设置缓存 1 小时
    response.newBuilder()
        .header("Cache-Control", "public, max-age=3600")
        .removeHeader("Pragma")
        .build()
}

// 3. 离线缓存（无网络时读缓存）
val offlineCacheInterceptor = Interceptor { chain ->
    var request = chain.request()
    if (!isNetworkAvailable()) {
        request = request.newBuilder()
            .header("Cache-Control", "public, only-if-cached, max-stale=86400")
            .build()
    }
    chain.proceed(request)
}

// 4. 网络状态监听
class NetworkMonitor(context: Context) {
    private val connectivityManager =
        context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager

    val isConnected: Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
        }

        connectivityManager.registerDefaultNetworkCallback(callback)
        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }
}
```

#### 核心要点

```
网络优化策略：
├─ 并行请求（coroutineScope + async）
├─ 请求缓存（OkHttp Cache）
├─ 数据压缩（GZIP，服务端支持）
├─ 连接复用（HTTP/2 多路复用）
├─ 减少请求次数（合并接口）
└─ 弱网适配（超时重试策略）
```

---

## 📌 数据存储

### 1. Android 存储方式对比

#### 详细答案

| 存储方式 | 适用场景 | 大小限制 | 跨进程 |
|---------|---------|---------|-------|
| **SharedPreferences** | 轻量 Key-Value 配置 | 小 | ❌ |
| **DataStore** | SP 的现代替代品 | 小 | ❌ |
| **SQLite/Room** | 结构化大量数据 | 无限制 | ❌ |
| **文件存储** | 图片、音频等 | 无限制 | 需 FileProvider |
| **ContentProvider** | 跨应用数据共享 | 无限制 | ✅ |

---

### 2. SharedPreferences 的线程安全问题

#### 详细答案

SharedPreferences 的 `apply()` 和 `commit()` 的区别：
- **commit()**：同步写入磁盘，返回成功/失败，在主线程调用会阻塞 UI
- **apply()**：异步写入磁盘，无返回值，不阻塞主线程（推荐）

**线程安全性：** SP 的读写是线程安全的（内部有锁），但在多进程下**不安全**。

#### 代码示例

```kotlin
// 1. 基本使用
class PreferencesManager(context: Context) {
    private val prefs = context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)

    fun saveToken(token: String) {
        prefs.edit().putString("token", token).apply()
    }

    fun getToken(): String? = prefs.getString("token", null)

    fun saveUser(userId: String, username: String) {
        prefs.edit()
            .putString("userId", userId)
            .putString("username", username)
            .putBoolean("isLoggedIn", true)
            .apply()
    }

    fun clearUser() {
        prefs.edit().clear().apply()
    }

    // 监听变化
    fun observeToken(listener: SharedPreferences.OnSharedPreferenceChangeListener) {
        prefs.registerOnSharedPreferenceChangeListener(listener)
    }
}

// 2. DataStore（推荐替代 SP）
class DataStoreManager(context: Context) {
    private val Context.dataStore: DataStore<Preferences> by preferencesDataStore("user_prefs")
    private val dataStore = context.dataStore

    companion object {
        val TOKEN_KEY = stringPreferencesKey("token")
        val USER_ID_KEY = stringPreferencesKey("userId")
        val IS_LOGGED_IN = booleanPreferencesKey("isLoggedIn")
    }

    // 读取（Flow，响应式）
    val token: Flow<String?> = dataStore.data.map { prefs ->
        prefs[TOKEN_KEY]
    }

    // 写入（协程，异步）
    suspend fun saveToken(token: String) {
        dataStore.edit { prefs ->
            prefs[TOKEN_KEY] = token
        }
    }

    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }
}
```

#### 核心要点

```
SP 问题：
├─ 多进程不安全（MODE_MULTI_PROCESS 已废弃）
├─ 大文件性能差（全量读写）
├─ apply() 可能导致 ANR（Activity 暂停时会等待写入）

DataStore 优势：
├─ 协程 + Flow，完全异步
├─ 类型安全（Proto DataStore）
├─ 原子操作，事务性更新
└─ 官方推荐替代 SharedPreferences
```

---

### 3. SQLite 事务和优化

#### 详细答案

SQLite 在大量数据操作时，使用**事务**可大幅提升性能（减少磁盘 I/O 次数）。

#### 代码示例

```java
// 1. 使用事务批量插入
public void insertWithTransaction(SQLiteDatabase db, List<User> users) {
    db.beginTransaction();
    try {
        for (User user : users) {
            ContentValues values = new ContentValues();
            values.put("name", user.getName());
            values.put("email", user.getEmail());
            db.insert("users", null, values);
        }
        db.setTransactionSuccessful();  // 标记事务成功
    } finally {
        db.endTransaction();  // 提交或回滚
    }
}

// 2. 使用 SQLiteStatement 预编译提高效率
public void batchInsert(SQLiteDatabase db, List<User> users) {
    SQLiteStatement stmt = db.compileStatement(
        "INSERT INTO users (name, email) VALUES (?, ?)"
    );

    db.beginTransaction();
    try {
        for (User user : users) {
            stmt.bindString(1, user.getName());
            stmt.bindString(2, user.getEmail());
            stmt.executeInsert();
            stmt.clearBindings();
        }
        db.setTransactionSuccessful();
    } finally {
        db.endTransaction();
    }
}

// 3. 建立索引提高查询速度
// CREATE INDEX idx_email ON users(email);

// 4. 优化查询
// ❌ SELECT * FROM users  -- 取所有列
// ✓ SELECT id, name FROM users  -- 只取需要的列
// ✓ 使用 LIMIT 分页：SELECT * FROM users LIMIT 20 OFFSET 40
// ✓ 使用 WHERE 过滤：SELECT * FROM users WHERE status = 1
```

---

### 4. ContentProvider 的使用

#### 详细答案

ContentProvider 是四大组件之一，用于**跨应用数据共享**，通过 `ContentResolver` 访问。

#### 代码示例

```java
// 1. 创建 ContentProvider
public class UserContentProvider extends ContentProvider {
    private static final String AUTHORITY = "com.example.provider";
    private static final Uri BASE_URI = Uri.parse("content://" + AUTHORITY);
    private static final Uri USERS_URI = Uri.withAppendedPath(BASE_URI, "users");

    private static final int USERS = 1;
    private static final int USER_ID = 2;

    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI(AUTHORITY, "users", USERS);
        uriMatcher.addURI(AUTHORITY, "users/#", USER_ID);
    }

    private UserDatabase database;

    @Override
    public boolean onCreate() {
        database = UserDatabase.getInstance(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        switch (uriMatcher.match(uri)) {
            case USERS:
                return database.query("users", projection, selection, selectionArgs, null, null, sortOrder);
            case USER_ID:
                String id = uri.getLastPathSegment();
                return database.query("users", projection, "id=?", new String[]{id}, null, null, null);
            default:
                throw new IllegalArgumentException("未知 URI: " + uri);
        }
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        long id = database.insert("users", null, values);
        // 通知数据变化
        getContext().getContentResolver().notifyChange(uri, null);
        return ContentUris.withAppendedId(USERS_URI, id);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        int count = database.update("users", values, selection, selectionArgs);
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        int count = database.delete("users", selection, selectionArgs);
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }

    @Override
    public String getType(Uri uri) {
        switch (uriMatcher.match(uri)) {
            case USERS: return "vnd.android.cursor.dir/vnd.example.users";
            case USER_ID: return "vnd.android.cursor.item/vnd.example.user";
            default: return null;
        }
    }
}

// 2. 使用 ContentResolver 访问
public class ContentResolverDemo {
    public void queryUsers(Context context) {
        Uri uri = Uri.parse("content://com.example.provider/users");
        Cursor cursor = context.getContentResolver().query(
            uri, null, null, null, null
        );

        if (cursor != null) {
            while (cursor.moveToNext()) {
                String name = cursor.getString(cursor.getColumnIndexOrThrow("name"));
                Log.d("ContentProvider", "用户: " + name);
            }
            cursor.close();
        }
    }

    // 监听数据变化
    public void observeUsers(Context context) {
        ContentObserver observer = new ContentObserver(new Handler()) {
            @Override
            public void onChange(boolean selfChange) {
                // 数据变化时调用
                Log.d("ContentProvider", "数据已更新");
            }
        };

        context.getContentResolver().registerContentObserver(
            Uri.parse("content://com.example.provider/users"),
            true,
            observer
        );
    }
}
```

#### 核心要点

```
ContentProvider 四个方法：
├─ query()：查询数据
├─ insert()：插入数据
├─ update()：更新数据
└─ delete()：删除数据

ContentResolver vs ContentProvider：
├─ ContentProvider：提供数据（在数据所在进程）
└─ ContentResolver：访问数据（在访问数据的进程）

URI 格式：
content://authority/path/id
├─ content://：协议
├─ authority：包名.provider
├─ path：表名
└─ id：数据 ID（可选）
```

---

## 📌 进程间通信（IPC）

### 1. Android 进程间通信方式

#### 详细答案

| 方式 | 使用难度 | 性能 | 数据大小 | 适用场景 |
|------|---------|------|---------|---------|
| **Bundle/Intent** | 简单 | 高 | < 1MB | Activity/Service 间简单数据 |
| **文件共享** | 简单 | 低 | 无限制 | 不要求实时，允许并发 |
| **SharedPreferences** | 简单 | 低 | 小 | 简单 Key-Value（不推荐多进程） |
| **Messenger** | 中等 | 中 | 中 | 低频率、无需并发的消息传递 |
| **AIDL** | 复杂 | 高 | 中 | 高频率、需要并发的跨进程调用 |
| **ContentProvider** | 中等 | 中 | 大 | 跨应用数据共享 |
| **Socket** | 中等 | 中 | 无限制 | 网络数据传输 |
| **Binder** | 复杂 | 高 | 中 | Android 系统底层 |

---

### 2. Binder 机制原理

#### 详细答案

Binder 是 Android 独有的 IPC 机制，基于**内存映射**（mmap），只需**一次数据拷贝**（其他 IPC 需要两次）。

**一次拷贝原理：**
```
传统 IPC（如 Socket/Pipe）：
用户空间A → 内核空间（copy_from_user）→ 内核空间（copy_to_user）→ 用户空间B
= 两次拷贝

Binder mmap：
内核空间 ←→ 接收进程用户空间（内存映射，共享物理内存）
发送进程 → 内核空间（一次拷贝）
接收进程 ← 内核空间（直接映射，0次拷贝）
= 一次拷贝
```

**Binder 四个角色：**
```
Client（客户端进程）
Server（服务端进程）
ServiceManager（名字服务，类似 DNS）
Binder Driver（内核驱动，/dev/binder）
```

#### 代码示例

```java
// Binder 底层使用 - 自定义 Binder（简化版）
public class LocalService extends Service {

    // 服务端 Binder 对象
    private final IUserManager.Stub binder = new IUserManager.Stub() {
        @Override
        public User getUser(String userId) throws RemoteException {
            // 在服务端进程执行
            return database.getUser(userId);
        }

        @Override
        public void addUser(User user) throws RemoteException {
            database.insertUser(user);
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}

// 客户端绑定服务
public class ClientActivity extends AppCompatActivity {
    private IUserManager userManager;
    private boolean isBound = false;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 获取远程 Binder 代理
            userManager = IUserManager.Stub.asInterface(service);
            isBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            userManager = null;
            isBound = false;
        }
    };

    @Override
    protected void onStart() {
        super.onStart();
        Intent intent = new Intent(this, LocalService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (isBound) {
            unbindService(connection);
            isBound = false;
        }
    }

    private void getUser(String userId) {
        if (isBound) {
            try {
                User user = userManager.getUser(userId);
                // 处理结果
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}
```

#### 核心要点

```
Binder 优势：
├─ 性能：只需一次数据拷贝（mmap 内存映射）
├─ 安全：内核层校验 UID/PID，无法伪造
└─ 稳定：C/S 架构，职责清晰

Binder 组成：
├─ IBinder：Binder 接口（远程调用能力）
├─ Binder：服务端实现
├─ BinderProxy：客户端代理
└─ AIDL：自动生成上面的代码
```

---

### 3. AIDL 的使用

#### 详细答案

AIDL（Android Interface Definition Language）是定义 Binder 跨进程接口的 IDL 语言。编译时自动生成 Stub（服务端）和 Proxy（客户端）代码。

#### 代码示例

```java
// 1. 定义 AIDL 接口（IUserManager.aidl）
// package com.example.aidl;
// import com.example.aidl.User;
// interface IUserManager {
//     User getUser(String userId);
//     void addUser(in User user);
//     List<User> getUserList();
// }

// 2. 定义 User.aidl（Parcelable 对象）
// package com.example.aidl;
// parcelable User;

// 3. User 实现 Parcelable
public class User implements Parcelable {
    private String id;
    private String name;

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }

    protected User(Parcel in) {
        id = in.readString();
        name = in.readString();
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(id);
        dest.writeString(name);
    }

    @Override
    public int describeContents() { return 0; }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) { return new User(in); }

        @Override
        public User[] newArray(int size) { return new User[size]; }
    };
}

// 4. 服务端 Service 实现
public class UserManagerService extends Service {
    // 数据
    private final CopyOnWriteArrayList<User> users = new CopyOnWriteArrayList<>();

    // AIDL Stub 实现（运行在 Binder 线程池中，注意线程安全）
    private final IUserManager.Stub stub = new IUserManager.Stub() {
        @Override
        public User getUser(String userId) throws RemoteException {
            for (User user : users) {
                if (user.getId().equals(userId)) {
                    return user;
                }
            }
            return null;
        }

        @Override
        public void addUser(User user) throws RemoteException {
            users.add(user);
        }

        @Override
        public List<User> getUserList() throws RemoteException {
            return new ArrayList<>(users);
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return stub;
    }
}
```

#### 核心要点

```
AIDL 定向 Tag（参数方向）：
├─ in：仅输入，数据从客户端到服务端
├─ out：仅输出，数据从服务端到客户端
└─ inout：双向传递

注意事项：
├─ AIDL 接口方法运行在 Binder 线程池（非主线程）
├─ 客户端调用可能阻塞（如果服务端执行耗时操作）
├─ 使用 CopyOnWriteArrayList 等线程安全集合
└─ 两端 .aidl 文件包名必须相同
```

---

### 4. Messenger 跨进程通信

#### 详细答案

Messenger 是对 AIDL 的封装，基于消息队列串行处理，不支持并发，比 AIDL 简单。

#### 代码示例

```java
// 服务端 Service
public class MessengerService extends Service {
    // 处理客户端消息的 Handler
    private static class IncomingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 1:  // 收到消息
                    String data = msg.getData().getString("data");
                    Log.d("Messenger", "收到: " + data);

                    // 回复客户端
                    Messenger client = msg.replyTo;
                    if (client != null) {
                        Message reply = Message.obtain(null, 2);
                        Bundle bundle = new Bundle();
                        bundle.putString("result", "处理成功: " + data);
                        reply.setData(bundle);
                        try {
                            client.send(reply);
                        } catch (RemoteException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
            }
        }
    }

    private final Messenger messenger = new Messenger(new IncomingHandler());

    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}

// 客户端 Activity
public class MessengerClientActivity extends AppCompatActivity {
    private Messenger serverMessenger;
    // 接收服务端回复的 Messenger
    private final Messenger clientMessenger = new Messenger(new Handler() {
        @Override
        public void handleMessage(Message msg) {
            if (msg.what == 2) {
                String result = msg.getData().getString("result");
                Log.d("Messenger", "服务端回复: " + result);
            }
        }
    });

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            serverMessenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            serverMessenger = null;
        }
    };

    private void sendMessage(String data) {
        if (serverMessenger == null) return;
        Message msg = Message.obtain(null, 1);
        Bundle bundle = new Bundle();
        bundle.putString("data", data);
        msg.setData(bundle);
        msg.replyTo = clientMessenger;  // 设置回复的 Messenger
        try {
            serverMessenger.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
}
```

#### 核心要点

```
Messenger vs AIDL：

Messenger：
├─ 基于 Handler + Message，串行处理
├─ 不支持并发（消息队列排队）
├─ 简单，适合低频消息传递
└─ 不能直接调用远程方法，只能发送消息

AIDL：
├─ 直接方法调用，支持并发
├─ 运行在 Binder 线程池
├─ 复杂，适合高频、需要返回值的场景
└─ 需要处理线程安全
```

---

### 5. Android 多进程的使用场景

#### 详细答案

在 AndroidManifest.xml 中通过 `android:process` 指定进程。

#### 代码示例

```xml
<!-- AndroidManifest.xml -->
<application>
    <!-- 主进程 -->
    <activity android:name=".MainActivity" />

    <!-- 独立进程（推送服务，防止主进程崩溃影响推送）-->
    <service
        android:name=".PushService"
        android:process=":push" />

    <!-- 独立进程（WebView，防止 WebView 崩溃影响主进程）-->
    <activity
        android:name=".WebViewActivity"
        android:process=":webview" />
</application>
```

```java
// 多进程中的问题处理
public class MultiProcessDemo extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        // 判断当前进程名
        String processName = getCurrentProcessName();

        if (processName.equals(getPackageName())) {
            // 主进程初始化
            initMainProcess();
        } else if (processName.endsWith(":push")) {
            // 推送进程初始化
            initPushProcess();
        }
    }

    private String getCurrentProcessName() {
        int pid = android.os.Process.myPid();
        ActivityManager am = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
        for (ActivityManager.RunningAppProcessInfo info : am.getRunningAppProcesses()) {
            if (info.pid == pid) {
                return info.processName;
            }
        }
        return "";
    }
}
```

#### 核心要点

```
多进程使用场景：
├─ 推送服务（防止主进程崩溃丢失推送）
├─ WebView（隔离崩溃风险）
├─ 大内存需求（每个进程独立内存）
└─ 音乐播放（后台进程持续运行）

多进程注意事项：
├─ Application 会多次创建（每个进程一个）
├─ 静态变量不共享（进程独立内存空间）
├─ SharedPreferences 多进程不安全
└─ 需要 IPC 通信（Binder/Messenger/ContentProvider）
```

---

### 6. Serializable 和 Parcelable 的区别

#### 详细答案

| 特性 | Serializable | Parcelable |
|------|-------------|-----------|
| **语言** | Java 标准 | Android 专有 |
| **性能** | 低（反射+IO） | 高（直接内存操作） |
| **实现** | 实现接口即可（简单） | 需要手写方法（繁琐） |
| **使用场景** | 持久化（文件/网络） | 内存中 IPC 传递 |

#### 代码示例

```java
// Serializable（简单）
public class UserSerializable implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int age;
    // 普通 Java 对象，无需额外实现
}

// Parcelable（高性能）
public class UserParcelable implements Parcelable {
    private String name;
    private int age;

    protected UserParcelable(Parcel in) {
        name = in.readString();
        age = in.readInt();
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(age);
    }

    @Override
    public int describeContents() { return 0; }

    public static final Creator<UserParcelable> CREATOR = new Creator<>() {
        @Override
        public UserParcelable createFromParcel(Parcel in) {
            return new UserParcelable(in);
        }

        @Override
        public UserParcelable[] newArray(int size) {
            return new UserParcelable[size];
        }
    };
}

// Kotlin 中使用 @Parcelize（推荐）
@Parcelize
data class UserKotlin(
    val name: String,
    val age: Int,
    val email: String
) : Parcelable
```

#### 核心要点

```
选择建议：
├─ 需要持久化（保存文件/数据库）→ Serializable
├─ Activity/Fragment 间传递数据 → Parcelable（推荐）
└─ Kotlin → @Parcelize（最简单）

为什么 Parcelable 比 Serializable 快？
├─ Serializable：使用反射读写字段 + 大量临时对象
└─ Parcelable：直接写入 Parcel 内存块，无反射，无 IO
```

---

### 7. Android 签名机制

#### 详细答案

Android 使用数字签名验证 APK 完整性和开发者身份。

**签名版本：**
- V1：基于 JAR 签名，逐文件校验（存在安全漏洞）
- V2（Android 7.0+）：对整个 APK 文件签名，更安全
- V3（Android 9.0+）：支持密钥轮替
- V4（Android 11+）：配合增量安装

```
签名流程：
开发者 → 生成密钥对（私钥+公钥）
开发者 → 用私钥对 APK 签名
发布到应用市场
用户安装 → 用公钥验证签名 → 验证 APK 未被篡改
```

#### 代码示例

```bash
# 生成密钥库
keytool -genkey -v -keystore my.keystore -alias mykey -keyalg RSA -keysize 2048 -validity 10000

# 对 APK 签名
jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore my.keystore app-release.apk mykey

# 使用 zipalign 优化
zipalign -v 4 app-release-unaligned.apk app-release.apk

# 验证签名
apksigner verify --verbose app-release.apk
```

```groovy
// build.gradle 配置签名
android {
    signingConfigs {
        release {
            storeFile file("my.keystore")
            storePassword "storePass"
            keyAlias "mykey"
            keyPassword "keyPass"
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

---

### 8. APK 构建流程

#### 详细答案

```
源码（Java/Kotlin）
    ↓ javac/kotlinc 编译
字节码（.class）
    ↓ D8/R8 编译（脱糖 + 混淆）
Dex 文件（.dex）

资源文件（res/）
    ↓ AAPT2 处理
编译后的资源（resources.arsc + R.java）

AndroidManifest.xml
    ↓ 合并（各 module）
合并后的 Manifest

Dex + 资源 + Manifest + Assets + JNI库（.so）
    ↓ APK 打包工具
未签名 APK
    ↓ apksigner 签名
已签名 APK
    ↓ zipalign 对齐
最终 APK
```

#### 核心要点

```
关键工具：
├─ AAPT2：资源编译打包
├─ D8/R8：字节码 → Dex（R8 同时做混淆优化）
├─ apksigner：APK 签名
└─ zipalign：4字节对齐（提高运行时内存访问速度）

R8 vs ProGuard：
├─ R8 是 Google 的 ProGuard 替代品
├─ R8：代码压缩 + 混淆 + 优化（3合1）
└─ 默认在 release 构建中启用（minifyEnabled = true）
```

---

## 📊 Part 3 完成总结

### ✅ 已完成（35 题）

**架构设计（10 题）：**
1. ✅ MVC / MVP / MVVM 对比
2. ✅ Jetpack ViewModel 原理
3. ✅ LiveData 工作原理
4. ✅ Room 数据库使用
5. ✅ Jetpack Navigation
6. ✅ DataBinding 使用和原理
7. ✅ WorkManager 使用场景
8. ✅ Paging 3 分页库
9. ✅ Glide 图片加载原理
10. ✅ MVVM + Clean Architecture

**网络通信（5 题）：**
11. ✅ HTTP vs HTTPS
12. ✅ OkHttp 原理和拦截器
13. ✅ Retrofit 原理
14. ✅ TCP/IP 三次握手四次挥手
15. ✅ 网络请求优化

**数据存储（4 题）：**
16. ✅ Android 存储方式对比
17. ✅ SharedPreferences 线程安全
18. ✅ SQLite 事务和优化
19. ✅ ContentProvider 使用

**进程间通信（8 题）：**
20. ✅ IPC 通信方式对比
21. ✅ Binder 机制原理
22. ✅ AIDL 使用
23. ✅ Messenger 跨进程通信
24. ✅ 多进程使用场景
25. ✅ Serializable vs Parcelable
26. ✅ Android 签名机制
27. ✅ APK 构建流程

---

## 📚 三部分汇总

| Part | 主题 | 题数 |
|------|------|------|
| **Part 1** | 四大组件 + 异步消息机制 | 22 题 |
| **Part 2** | Android UI 绘制 + 性能优化 | 45 题 |
| **Part 3** | 架构设计 + 网络 + 存储 + IPC | 35 题 |
| **合计** | | **102 题** |

---

**是否继续生成 Part 4？** （Kotlin + 数据结构 + 设计模式）🚀