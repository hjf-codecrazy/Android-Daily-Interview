# Android 技术面试题（基于简历定制）

> 难度：🟢 基础 / 🟡 中级 / 🔴 进阶

---

## 一、Android 核心架构

### 1. MVVM 架构

**Q1** 🟢 请描述 MVVM 各层的职责，以及你项目中是如何分层的？

> **答题要点：**
> - Model：数据层（Repository + 数据源，本地DB/远程API）
> - ViewModel：持有 UI 状态，处理业务逻辑，不持有 View 引用
> - View：观察 LiveData/StateFlow，只负责渲染
> - Repository 模式：屏蔽数据来源，ViewModel 不关心数据从哪来
> - 结合项目：YLS 中 ViewModel 持有健康数据 StateFlow，Fragment 订阅更新图表

```kotlin
// Repository：屏蔽数据来源
class HealthRepository(
    private val remoteApi: HealthApi,
    private val localDao: HealthDao
) {
    fun getHeartRateFlow(): Flow<HeartRate> = localDao.observeLatest()

    suspend fun syncFromServer() {
        val data = remoteApi.fetchHealthData()
        localDao.insertAll(data)
    }
}

// ViewModel：持有状态，不引用 View
class HealthViewModel(private val repo: HealthRepository) : ViewModel() {
    val heartRate: StateFlow<HeartRate?> = repo.getHeartRateFlow()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
}

// Fragment：只负责订阅和渲染
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.heartRate.collect { rate ->
            binding.tvHeartRate.text = "${rate?.bpm} bpm"
        }
    }
}
```

---

**Q2** 🟡 ViewModel 为什么能在 Activity 旋转后保留数据？

> **答题要点：**
> - ViewModel 由 `ViewModelStore` 持有，Activity 旋转时系统重建 Activity 但保留 ViewModelStore
> - ViewModel 生命周期绑定到 ViewModelStoreOwner（Activity/Fragment）的生命周期，直到 `onCleared()` 真正销毁
> - 注意：进程被杀后 ViewModel 不保留，需配合 `SavedStateHandle`

```kotlin
// 普通 ViewModel：旋转后数据保留，进程重启后丢失
class CounterViewModel : ViewModel() {
    var count = 0
}

// 配合 SavedStateHandle：进程重启后也能恢复
class CounterViewModel(private val state: SavedStateHandle) : ViewModel() {
    // 进程被杀重启后，count 依然从 state 中恢复
    var count: Int
        get() = state["count"] ?: 0
        set(value) { state["count"] = value }
}

// Activity 中使用（旋转10次，ViewModel 实例只创建1次）
class MainActivity : AppCompatActivity() {
    private val viewModel: CounterViewModel by viewModels()
}
```

---

**Q3** 🔴 LiveData 和 StateFlow 的区别？你在项目里怎么选择？

> **答题要点：**
> - LiveData：生命周期感知，自动取消订阅，粘性事件（新观察者会收到最新值）
> - StateFlow：需手动处理生命周期（`lifecycleScope.repeatOnLifecycle`），初始值必须有，适合状态
> - SharedFlow：无初始值，适合一次性事件（Toast、导航）
> - 选择原则：UI 状态 → StateFlow；一次性事件 → SharedFlow；纯 UI 更新且不需要协程 → LiveData
> - 项目中：YLS 健康数据用 StateFlow，网络错误提示用 SharedFlow

```kotlin
class HealthViewModel : ViewModel() {

    // ✅ UI 状态用 StateFlow（有初始值，可以被多次观察）
    private val _uiState = MutableStateFlow<HealthUiState>(HealthUiState.Loading)
    val uiState: StateFlow<HealthUiState> = _uiState.asStateFlow()

    // ✅ 一次性事件用 SharedFlow（Toast/导航，不需要初始值）
    private val _event = MutableSharedFlow<UiEvent>()
    val event: SharedFlow<UiEvent> = _event.asSharedFlow()

    fun loadData() {
        viewModelScope.launch {
            try {
                val data = repository.fetchHealth()
                _uiState.value = HealthUiState.Success(data)
            } catch (e: Exception) {
                _event.emit(UiEvent.ShowToast("加载失败：${e.message}"))
            }
        }
    }
}

// Fragment 订阅
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        // 状态：每次回到前台都能拿到最新值
        launch { viewModel.uiState.collect { render(it) } }
        // 事件：每个 Toast 只弹一次
        launch { viewModel.event.collect { handleEvent(it) } }
    }
}
```

---

### 2. Kotlin 协程 & Flow

**Q4** 🟢 协程和线程的区别？协程是轻量级的原因是什么？

> **答题要点：**
> - 线程：OS 级别，切换需要内核参与，栈空间固定（默认 1MB+）
> - 协程：用户态调度，挂起/恢复不需要 OS 介入，挂起时只保存极少状态
> - 轻量：协程挂起时不阻塞线程，一个线程可以运行成千上万协程
> - 本质：编译器将 suspend 函数转成状态机（CPS 变换）

```kotlin
// 线程：每个线程占用约 1MB 栈内存，1万个线程 ≈ 10GB
repeat(10_000) {
    Thread { Thread.sleep(1000) }.start() // 可能 OOM
}

// 协程：挂起时不占线程，1万个协程轻松运行
runBlocking {
    repeat(10_000) {
        launch { delay(1000) } // delay 挂起但不阻塞线程
    }
}

// suspend 函数编译后的状态机（简化版）
// 原始代码：
suspend fun fetchUser(): User {
    val token = getToken()   // 挂起点1
    val user = getUser(token) // 挂起点2
    return user
}
// 编译器转成状态机，label 记录当前位置，挂起时保存 label 和局部变量
```

---

**Q5** 🟡 `Flow` 的冷流和热流有什么区别？举例说明。

> **答题要点：**
> - 冷流（Cold）：每次 collect 都会重新执行，`flow { }` 构建的是冷流
> - 热流（Hot）：不管有没有订阅者都在运行，`StateFlow`、`SharedFlow` 是热流
> - 场景：网络请求用冷流（按需触发）；传感器数据、蓝牙数据用热流（持续推送）
> - YLS 中蓝牙心率数据：Ring SDK 回调 → `MutableSharedFlow.emit()` → UI 订阅

```kotlin
// 冷流：每次 collect 重新执行，互相独立
val coldFlow = flow {
    println("开始请求网络") // 每次 collect 都会打印
    emit(fetchApi())
}
coldFlow.collect { } // 打印"开始请求网络"
coldFlow.collect { } // 再次打印"开始请求网络"

// 热流：SharedFlow 持续发射，多个订阅者共享同一数据流
class BleService {
    // 蓝牙数据热流，replay=1 让新订阅者收到最新心率
    val heartRateFlow = MutableSharedFlow<Int>(replay = 1)

    // BLE 回调中 emit（不管有没有 UI 订阅，数据都在流动）
    private val gattCallback = object : BluetoothGattCallback() {
        override fun onCharacteristicChanged(gatt: BluetoothGatt, char: BluetoothGattCharacteristic) {
            val bpm = char.value[1].toInt()
            serviceScope.launch { heartRateFlow.emit(bpm) }
        }
    }
}

// UI 订阅（两个页面订阅同一热流，数据只产生一份）
viewModel.heartRate.collect { bpm -> updateUI(bpm) }
```

---

**Q6** 🔴 `launchIn`、`lifecycleScope.launch`、`repeatOnLifecycle` 的区别和使用场景？

> **答题要点：**
> - `lifecycleScope.launch`：在 DESTROYED 才取消，页面不可见时仍然收集，浪费资源
> - `launchIn(lifecycleScope)`：同上，只是写法不同
> - `repeatOnLifecycle(Lifecycle.State.STARTED)`：在 STARTED 时收集，STOPPED 时自动取消，页面回到前台重新订阅
> - **结论**：UI 相关的 Flow 订阅必须用 `repeatOnLifecycle`，否则在后台也会更新 UI 可能崩溃

```kotlin
// ❌ 错误方式：页面进后台后仍然收集，更新 UI 可能崩溃或浪费资源
lifecycleScope.launch {
    viewModel.uiState.collect { render(it) }
}

// ❌ 同样错误：launchIn 只是 collect { }.launchIn() 的语法糖
viewModel.uiState.onEach { render(it) }.launchIn(lifecycleScope)

// ✅ 正确方式：STARTED 时收集，进后台（STOPPED）自动暂停
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { render(it) }
    }
}

// ✅ 多个 Flow 同时订阅
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        launch { viewModel.uiState.collect { render(it) } }
        launch { viewModel.event.collect { handleEvent(it) } }
    }
}
```

---

### 3. Jetpack 组件

**Q7** 🟡 Room 数据库升级时如何处理迁移？如果字段变更怎么办？

> **答题要点：**
> - 提供 `Migration(oldVersion, newVersion)` 编写 SQL
> - `fallbackToDestructiveMigration()` 开发期可用，生产环境不能用（数据丢失）
> - 字段新增：`ALTER TABLE ... ADD COLUMN`
> - 字段删除/修改：创建新表 → 复制数据 → 删旧表 → 重命名
> - 建议：结合 `@Database(exportSchema = true)` 导出 schema 文件做版本追踪

```kotlin
// 版本1 → 版本2：新增字段
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE health_data ADD COLUMN sleep_score INTEGER NOT NULL DEFAULT 0")
    }
}

// 版本2 → 版本3：修改字段（Room 不支持直接 ALTER COLUMN，需重建表）
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // 1. 创建新表
        database.execSQL("""
            CREATE TABLE health_data_new (
                id INTEGER PRIMARY KEY NOT NULL,
                bpm INTEGER NOT NULL,
                sleep_score REAL NOT NULL DEFAULT 0.0  -- 类型从 INTEGER 改为 REAL
            )
        """)
        // 2. 复制数据
        database.execSQL("INSERT INTO health_data_new SELECT id, bpm, sleep_score FROM health_data")
        // 3. 删旧表，重命名
        database.execSQL("DROP TABLE health_data")
        database.execSQL("ALTER TABLE health_data_new RENAME TO health_data")
    }
}

// 注册迁移
Room.databaseBuilder(context, AppDatabase::class.java, "health.db")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

---

## 二、蓝牙 / BLE 开发

**Q8** 🟢 BLE 和经典蓝牙的区别？戒指类设备为什么用 BLE？

> **答题要点：**
> - 经典蓝牙：高带宽（音频/文件传输），功耗高
> - BLE：低功耗（可用纽扣电池运行数月），适合小数据量周期上报
> - 戒指原因：电池极小，健康数据（心率、步数）数据量小，BLE 完全够用

```
经典蓝牙 vs BLE 对比：
┌─────────────┬──────────────┬─────────────────┐
│    特性      │   经典蓝牙   │      BLE        │
├─────────────┼──────────────┼─────────────────┤
│  功耗        │  高（持续）  │  极低（可休眠） │
│  带宽        │  2-3 Mbps   │  125Kbps-2Mbps  │
│  连接时间    │  100ms       │  3ms 以内       │
│  典型应用    │  耳机、音箱  │  手环、戒指、心率计 │
│  电池续航    │  数小时      │  数月至数年     │
└─────────────┴──────────────┴─────────────────┘
```

---

**Q9** 🟡 描述 Android BLE 连接的完整流程（从扫描到数据读取）

> **答题要点：**
> 1. 权限申请：Android 12+ 需要 `BLUETOOTH_SCAN`、`BLUETOOTH_CONNECT`
> 2. 扫描：`BluetoothLeScanner.startScan()`，通过 `ScanFilter` 过滤设备
> 3. 连接：`device.connectGatt(context, autoConnect, gattCallback)`
> 4. 发现服务：`onConnectionStateChange` 成功后调用 `gatt.discoverServices()`
> 5. 操作 Characteristic：`onServicesDiscovered` 后找到目标 Service → Characteristic
> 6. 读取：`gatt.readCharacteristic(char)` → `onCharacteristicRead` 回调
> 7. 订阅通知：`gatt.setCharacteristicNotification(char, true)` + 写 CCCD Descriptor

```kotlin
// 完整 BLE 连接流程
class BleManager(private val context: Context) {

    private var bluetoothGatt: BluetoothGatt? = null

    // 1. 扫描（过滤特定设备名）
    fun startScan() {
        val filter = ScanFilter.Builder().setDeviceName("YLS-Ring").build()
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY).build()
        scanner.startScan(listOf(filter), settings, scanCallback)
    }

    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            // 2. 找到设备，停止扫描并连接
            scanner.stopScan(this)
            connect(result.device)
        }
    }

    // 3. 建立 GATT 连接（autoConnect=false 更快）
    fun connect(device: BluetoothDevice) {
        bluetoothGatt = device.connectGatt(context, false, gattCallback)
    }

    private val gattCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
            if (newState == BluetoothProfile.STATE_CONNECTED) {
                // 4. 连接成功，发现服务
                gatt.discoverServices()
            }
        }

        override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
            // 5. 找到心率 Characteristic（标准 UUID）
            val heartRateChar = gatt
                .getService(UUID.fromString("0000180d-..."))
                ?.getCharacteristic(UUID.fromString("00002a37-..."))

            // 7. 开启通知（订阅心率实时推送）
            heartRateChar?.let { char ->
                gatt.setCharacteristicNotification(char, true)
                // 写入 CCCD Descriptor，告知设备开始推送
                val descriptor = char.getDescriptor(UUID.fromString("00002902-..."))
                descriptor.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
                gatt.writeDescriptor(descriptor)
            }
        }

        override fun onCharacteristicChanged(gatt: BluetoothGatt, char: BluetoothGattCharacteristic) {
            // 持续收到心率数据
            val bpm = char.value[1].toInt()
            scope.launch { heartRateFlow.emit(bpm) }
        }
    }
}
```

---

**Q10** 🔴 BLE 连接断开（如戒指离手机太远）如何实现自动重连？有什么坑？

> **答题要点：**
> - 监听 `onConnectionStateChange` 中 `newState == STATE_DISCONNECTED`
> - 重连策略：指数退避（1s → 2s → 4s → ... → 最大60s）
> - 坑1：`autoConnect=true` 系统重连很慢（可能几分钟）
> - 坑2：Android 部分厂商（华为/小米）在后台会限制扫描频率
> - 坑3：`gatt.connect()` 重连前必须先 `gatt.close()`，否则会有 133 错误
> - 坑4：同一设备重复 `connectGatt` 会创建多个 GATT 对象，导致资源泄漏
> - 最佳实践：用 Service 在后台维持连接，前台通知保活

```kotlin
// 指数退避重连策略
class BleReconnectManager {

    private var retryCount = 0
    private val maxRetryDelay = 60_000L // 最大 60s
    private var reconnectJob: Job? = null

    fun onDisconnected(gatt: BluetoothGatt, device: BluetoothDevice) {
        // ⚠️ 坑3：必须先 close，否则再连接会报 133 错误
        gatt.close()
        bluetoothGatt = null

        scheduleReconnect(device)
    }

    private fun scheduleReconnect(device: BluetoothDevice) {
        reconnectJob?.cancel()
        reconnectJob = scope.launch {
            // 指数退避：1s, 2s, 4s, 8s, ... 最大 60s
            val delay = minOf(1000L * (1 shl retryCount), maxRetryDelay)
            delay(delay)
            retryCount++

            // ⚠️ 坑4：确保没有残留的 GATT 连接
            connect(device)
        }
    }

    private fun connect(device: BluetoothDevice) {
        // ⚠️ autoConnect=false：更快，失败后自己控制重试
        // ⚠️ autoConnect=true：系统慢慢重连，不可控
        bluetoothGatt = device.connectGatt(context, false, gattCallback)
    }

    // 连接成功后重置计数
    fun onConnected() {
        retryCount = 0
        reconnectJob?.cancel()
    }
}
```

---

**Q11** 🔴 BLE 的 MTU 是什么？如何传输大数据包？

> **答题要点：**
> - MTU（Maximum Transmission Unit）：单次传输最大字节数，默认 23 字节（实际数据 20 字节）
> - 协商：`gatt.requestMtu(512)` → `onMtuChanged` 回调实际值
> - 大数据传输（如固件升级 OTA）：手动分包，每包不超过 MTU-3 字节
> - 注意：部分设备不支持大 MTU，需要做降级处理

```kotlin
// MTU 协商（连接成功后立即协商）
override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
    if (newState == BluetoothProfile.STATE_CONNECTED) {
        gatt.requestMtu(512) // 请求最大 MTU
    }
}

override fun onMtuChanged(gatt: BluetoothGatt, mtu: Int, status: Int) {
    // mtu 是实际协商结果（可能小于请求值）
    // 实际可用数据大小 = mtu - 3（ATT 协议开销）
    val maxPacketSize = mtu - 3
    startOtaUpdate(maxPacketSize)
}

// 大数据分包发送（OTA 固件升级场景）
fun sendLargeData(data: ByteArray, maxPacketSize: Int) {
    scope.launch {
        data.toList()
            .chunked(maxPacketSize) // 按 MTU 切片
            .forEachIndexed { index, chunk ->
                val packet = chunk.toByteArray()
                writeCharacteristic(packet)
                // 等待写入确认（onCharacteristicWrite 回调）再发下一包
                writeAckChannel.receive()
                // 上报进度
                val progress = (index + 1) * maxPacketSize * 100 / data.size
                _otaProgress.value = progress.coerceAtMost(100)
            }
    }
}
```

---

## 三、性能优化

**Q12** 🟢 ANR 的触发条件是什么？如何定位和排查？

> **答题要点：**
> - 触发条件：主线程 5s 内无响应（按键/触摸）；BroadcastReceiver 前台10s/后台60s；Service 前台20s/后台200s
> - 排查：`/data/anr/traces.txt`，查看主线程堆栈
> - 常见原因：主线程执行 IO/网络/锁等待、死锁、Binder 调用耗时
> - 线上监控：`BlockCanary`、`ANR-WatchDog`、`Matrix`（腾讯）
> - 预防：`StrictMode` 开发期检测主线程 IO

```kotlin
// ❌ 触发 ANR：主线程做网络/IO
fun onClick() {
    val data = URL("https://api.example.com").readText() // 主线程网络，必然 ANR
    tvResult.text = data
}

// ✅ 正确：切到 IO 线程，结果回主线程
fun onClick() {
    lifecycleScope.launch {
        val data = withContext(Dispatchers.IO) {
            URL("https://api.example.com").readText()
        }
        tvResult.text = data // 自动回到主线程
    }
}

// 开发期开启 StrictMode 检测主线程违规
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .penaltyLog()      // 打印日志
                    .penaltyDialog()   // 弹出对话框（开发期快速发现）
                    .build()
            )
        }
    }
}
```

---

**Q13** 🟡 OOM 常见场景和解决方案？

> **答题要点：**
> - 场景1：Bitmap 未压缩直接加载大图 → 使用 Glide/Coil，设置 `override()` 尺寸
> - 场景2：内存泄漏积累（Activity 被静态持有）→ LeakCanary 检测
> - 场景3：大量对象频繁创建触发 GC → 对象池复用
> - 场景4：WebView 导致内存泄漏 → 在独立进程中运行 WebView
> - 分析工具：Android Profiler Memory、MAT（Memory Analyzer Tool）
> - Bitmap 优化：`inSampleSize` 降采样、`RGB_565` 替代 `ARGB_8888`（内存减半）

```kotlin
// ❌ 直接加载原图（4000x3000 ARGB_8888 ≈ 46MB，极易 OOM）
val bitmap = BitmapFactory.decodeFile(path)
imageView.setImageBitmap(bitmap)

// ✅ 方式1：Glide 自动处理（推荐）
Glide.with(this)
    .load(path)
    .override(800, 600) // 限制解码尺寸，节省内存
    .into(imageView)

// ✅ 方式2：手动 inSampleSize 降采样
fun decodeSampledBitmap(path: String, reqWidth: Int, reqHeight: Int): Bitmap {
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true // 只读尺寸，不加载像素
        BitmapFactory.decodeFile(path, this)
        inSampleSize = calculateInSampleSize(this, reqWidth, reqHeight)
        inJustDecodeBounds = false
        inPreferredConfig = Bitmap.Config.RGB_565 // ARGB_8888 内存的一半
    }
    return BitmapFactory.decodeFile(path, options)
}

// ❌ 内存泄漏：静态持有 Activity
object Cache {
    var context: Context? = null // Activity 永远无法被 GC
}

// ✅ 使用 ApplicationContext 或 WeakReference
object Cache {
    lateinit var appContext: Context // Application 级别，不会泄漏
}
```

---

**Q14** 🔴 Android 15 的 16KB Page Size 适配，你做了哪些工作？

> **答题要点：**
> - 背景：Android 15 起支持 16KB 内存页（原来 4KB），提升性能但不兼容旧 .so
> - 需要适配的：所有 native .so 库必须按 16KB 对齐编译
> - 检测方法：`adb shell /system/bin/linker_config_checker` 或 Android Studio 提示
> - 编译修改：CMakeLists.txt 中 `target_link_options(... -Wl,-z,max-page-size=16384)`
> - 三方库：需要等待三方库更新，或自行重编译
> - 验证：在 Android 15 模拟器（16KB模式）运行测试

```cmake
# CMakeLists.txt 适配 16KB Page Size
cmake_minimum_required(VERSION 3.22.1)
project("myapp")

add_library(mylib SHARED mylib.cpp)

# 关键：添加链接选项，支持最大 16KB 内存页对齐
target_link_options(mylib PRIVATE "-Wl,-z,max-page-size=16384")
```

```bash
# 检测 .so 文件是否已对齐
# 使用 Android SDK 工具检查
python3 $ANDROID_SDK/tools/check_elf_alignment.py mylib.so

# 或用 readelf 查看 LOAD segment 对齐值（需要是 0x4000 = 16384）
readelf -l mylib.so | grep -A1 LOAD
# 期望输出：Align: 0x4000
```

```kotlin
// build.gradle 中确认 NDK 版本（r27+ 默认支持 16KB 对齐）
android {
    defaultConfig {
        externalNativeBuild {
            cmake {
                arguments "-DANDROID_SUPPORT_FLEXIBLE_PAGE_SIZES=ON"
            }
        }
    }
}
```

---

**Q15** 🟡 Glide 的缓存机制是怎样的？

> **答题要点：**
> - 四级缓存：活跃缓存（弱引用，当前显示）→ 内存缓存（LruCache）→ 磁盘缓存 → 网络
> - 磁盘缓存两种：`DATA`（原始数据）和 `RESOURCE`（解码后的 Bitmap）
> - 缓存 Key：由 URL + 宽高 + 变换等组合生成
> - 跳过缓存：`.skipMemoryCache(true).diskCacheStrategy(DiskCacheStrategy.NONE)`

```kotlin
// 默认加载（走完整缓存链）
Glide.with(this).load(url).into(imageView)

// 跳过内存缓存（如头像实时更新场景）
Glide.with(this)
    .load(url)
    .skipMemoryCache(true)
    .diskCacheStrategy(DiskCacheStrategy.NONE)
    .into(imageView)

// 只缓存原始数据（节省磁盘，下次仍需解码）
Glide.with(this)
    .load(url)
    .diskCacheStrategy(DiskCacheStrategy.DATA)
    .into(imageView)

// 只缓存解码后资源（加载更快，但磁盘占用更大）
Glide.with(this)
    .load(url)
    .diskCacheStrategy(DiskCacheStrategy.RESOURCE)
    .into(imageView)

/*
Glide 缓存查找顺序：
1. 活跃缓存（WeakReference）：当前正在展示的图片，命中直接返回
2. 内存缓存（LruCache）：最近使用的图片，命中从 LRU 移到活跃缓存
3. 磁盘缓存（DiskLruCache）：命中后解码，存入活跃缓存
4. 网络请求：下载 → 存磁盘 → 解码 → 存活跃缓存
*/
```

---

## 四、网络层

**Q16** 🟢 OkHttp 的拦截器链是怎么工作的？

> **答题要点：**
> - 责任链模式：`Chain.proceed()` 将请求传递给下一个拦截器
> - 顺序（从外到内）：自定义拦截器 → RetryAndFollowUp → Bridge → Cache → Connect → Network
> - 应用拦截器（`addInterceptor`）：不管缓存，只调用一次，适合日志/全局Header
> - 网络拦截器（`addNetworkInterceptor`）：可看到实际网络请求，重定向时会多次调用
> - 项目实践：添加 Token 拦截器、日志拦截器、错误统一处理拦截器

```kotlin
// Token 注入拦截器（应用拦截器）
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer ${tokenProvider.getToken()}")
            .build()
        return chain.proceed(request) // 传给下一个拦截器
    }
}

// 全局错误处理拦截器
class ErrorInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        when (response.code) {
            401 -> throw UnauthorizedException()
            403 -> throw ForbiddenException()
            500 -> throw ServerException()
        }
        return response
    }
}

// 组装 OkHttpClient
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenProvider))  // 应用拦截器：加 Token
    .addInterceptor(HttpLoggingInterceptor())        // 应用拦截器：日志
    .addInterceptor(ErrorInterceptor())              // 应用拦截器：统一错误处理
    .connectTimeout(30, TimeUnit.SECONDS)
    .build()
```

---

**Q17** 🟡 Retrofit 是如何通过注解生成接口实现的？

> **答题要点：**
> - 动态代理：`Retrofit.create(ApiService::class.java)` 返回代理对象
> - 调用方法时，`InvocationHandler` 拦截，解析方法上的注解（@GET/@POST/@Path等）
> - 构建 `ServiceMethod`（含 OkHttp Request 构建逻辑）
> - 通过 `CallAdapter` 转换返回类型（Call → Deferred → Flow）
> - 通过 `Converter` 转换请求/响应体（Gson/Moshi）

```kotlin
// 接口定义（注解描述请求）
interface HealthApi {
    @GET("users/{userId}/health")
    suspend fun getHealthData(@Path("userId") userId: String): HealthResponse

    @POST("health/sync")
    suspend fun syncData(@Body data: HealthData): SyncResult
}

// Retrofit 构建
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.yls.com/v1/")
    .client(okHttpClient)
    .addConverterFactory(GsonConverterFactory.create())      // 处理 JSON 序列化
    .addCallAdapterFactory(CoroutineCallAdapterFactory())   // 支持 suspend 函数
    .build()

// create() 内部原理（简化）：
// val api = retrofit.create(HealthApi::class.java)
// 等价于：
// val api = Proxy.newProxyInstance(loader, interfaces) { proxy, method, args ->
//     val serviceMethod = loadServiceMethod(method) // 解析注解，构建 Request
//     serviceMethod.invoke(args)                    // 用 OkHttp 执行请求
// }
val api = retrofit.create(HealthApi::class.java)
val data = api.getHealthData("user123") // 实际执行网络请求
```

---

**Q18** 🔴 如何实现 Token 过期自动刷新，且多个并发请求只刷新一次？

> **答题要点：**
> - 在 OkHttp 拦截器中处理 401 响应
> - 多并发问题：用 `Mutex`（协程）或 `synchronized` 加锁
> - 思路：第一个线程触发 refresh，其余线程挂起等待；refresh 完成后所有线程用新 token 重试
> - 代码关键：`@Volatile var isRefreshing` + 双重检查，或协程 `Mutex.withLock { }`
> - 注意：refresh 本身的请求不能再经过这个拦截器（避免死循环）

```kotlin
class TokenRefreshInterceptor(
    private val tokenStore: TokenStore,
    private val authApi: AuthApi
) : Interceptor {

    // Mutex 保证多个并发请求只触发一次 refresh
    private val refreshMutex = Mutex()

    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(addToken(chain.request()))

        if (response.code != 401) return response

        // 401：需要刷新 Token
        return runBlocking {
            // withLock 保证：同时有 10 个请求失败，只有 1 个执行 refresh
            // 其余 9 个挂起等待，refresh 完成后用新 token 重试
            refreshMutex.withLock {
                val currentToken = tokenStore.getToken()
                val requestToken = chain.request().header("Authorization")

                // 双重检查：如果 token 已被别的请求刷新过，直接用新 token 重试
                if ("Bearer $currentToken" != requestToken) {
                    return@withLock chain.proceed(addToken(chain.request()))
                }

                // 执行 refresh（此请求需绕过本拦截器，避免死循环）
                val newToken = authApi.refreshToken(tokenStore.getRefreshToken())
                tokenStore.saveToken(newToken)

                // 用新 token 重试原始请求
                chain.proceed(addToken(chain.request()))
            }
        }
    }

    private fun addToken(request: Request) = request.newBuilder()
        .header("Authorization", "Bearer ${tokenStore.getToken()}")
        .build()
}
```

---

## 五、Web3 / 区块链

**Q19** 🟢 你在项目中接入 WalletConnect 的流程是怎样的？

> **答题要点：**
> - WalletConnect v2 基于 WebSocket 中继服务器，App 和钱包通过 topic 通信
> - 流程：App 生成 URI → 用户用钱包扫码 → 建立会话（Session）→ 发起请求（签名/交易）→ 钱包确认
> - Android SDK：`WalletConnect` 官方 Kotlin SDK
> - 注意：URI 有时效性，session 过期需要重新连接

```kotlin
// WalletConnect v2 接入流程
class WalletConnectManager {

    // 1. 初始化（Application 中）
    fun init(context: Context) {
        val initParams = Core.Params.Init(
            application = context as Application,
            relayServerUrl = "wss://relay.walletconnect.com?projectId=YOUR_PROJECT_ID"
        )
        CoreClient.initialize(initParams)
        Web3Wallet.initialize(Wallet.Params.Init(core = CoreClient))
    }

    // 2. 生成配对 URI，展示二维码给用户扫描
    fun generateUri(): String {
        val pairingParams = Core.Params.Pair("wc:...") // 或创建新 pairing
        CoreClient.Pairing.create { pairing ->
            val uri = pairing.uri // 展示此 URI 的二维码
            showQrCode(uri)
        }
        return ""
    }

    // 3. 监听会话建立
    fun observeSession() {
        Web3Wallet.setWalletDelegate(object : Web3Wallet.WalletDelegate {
            override fun onSessionProposal(proposal: Wallet.Model.SessionProposal) {
                // 用户扫码后，显示授权界面
                showApproveDialog(proposal)
            }
            override fun onSessionRequest(request: Wallet.Model.SessionRequest) {
                // 用户在 DApp 发起签名/交易请求
                handleSignRequest(request)
            }
        })
    }
}
```

---

**Q20** 🟡 web3j 调用智能合约的方式有哪些？load 和 deploy 的区别？

> **答题要点：**
> - `Contract.deploy()`：部署新合约到链上，返回合约地址
> - `Contract.load(address, ...)`：加载已部署合约，通过地址交互
> - 调用只读方法（`view/pure`）：`call()`，不消耗 Gas，不上链
> - 调用写入方法：`send()`，需要签名和 Gas，等待 TransactionReceipt
> - ABI：合约的接口描述文件，web3j 通过 ABI 生成 Java/Kotlin Wrapper 类

```kotlin
// web3j 连接以太坊节点
val web3j = Web3j.build(HttpService("https://mainnet.infura.io/v3/YOUR_KEY"))
val credentials = Credentials.create("0x私钥")

// 加载已部署合约（常用）
val contract = MyNFT.load(
    contractAddress,   // 合约地址
    web3j,
    credentials,
    DefaultGasProvider()
)

// 调用只读方法（view）：不消耗 Gas，立即返回
val totalSupply: BigInteger = contract.totalSupply().send() // call()
val owner: String = contract.ownerOf(BigInteger.ONE).send()

// 调用写入方法（需要签名 + Gas）
val receipt: TransactionReceipt = contract.mint(
    recipientAddress,
    tokenId
).send() // 等待交易上链（可能需要数秒到数分钟）

println("交易 Hash: ${receipt.transactionHash}")
println("是否成功: ${receipt.isStatusOK}")
```

---

**Q21** 🔴 NFT 铸造的链上流程是什么？你是如何处理交易失败和 Gas 估算的？

> **答题要点：**
> - 铸造：调用合约 `mint(to, tokenId)` 方法，写入链上
> - Gas 估算：`web3j.ethEstimateGas(transaction).send()`，实际设置时乘以 1.2 留余量
> - Gas Price：`ethGasPrice` 获取当前网络建议价格，或用 EIP-1559（maxFeePerGas）
> - 失败处理：监听 TransactionReceipt 的 `status`（"0x1" 成功，"0x0" 失败）
> - 超时处理：设置超时，超时后查询 txHash 状态，而不是无限等待
> - 用户体验：铸造过程中显示进度，Hash 生成后即可给用户看（乐观更新）

```kotlin
suspend fun mintNFT(toAddress: String, tokenId: BigInteger): MintResult {
    // 1. 估算 Gas（乘以 1.2 留余量）
    val encodedFunction = contract.mint(toAddress, tokenId).encodeFunctionCall()
    val estimatedGas = web3j.ethEstimateGas(
        Transaction.createEthCallTransaction(credentials.address, contractAddress, encodedFunction)
    ).send().amountUsed
    val gasLimit = estimatedGas.multiply(BigInteger.valueOf(12)).divide(BigInteger.TEN) // *1.2

    // 2. 获取当前 Gas Price（EIP-1559）
    val feeHistory = web3j.ethFeeHistory(1, "latest", null).send()
    val baseFee = feeHistory.feeHistory.baseFeePerGas.last()
    val maxPriorityFee = BigInteger.valueOf(2_000_000_000L) // 2 Gwei 小费
    val maxFeePerGas = baseFee.multiply(BigInteger.TWO).add(maxPriorityFee)

    // 3. 发送交易（立即获得 txHash，UI 乐观更新）
    val txHash = contract.mint(toAddress, tokenId).sendAsync().get().transactionHash
    _mintState.value = MintState.Pending(txHash) // 告知用户交易已提交

    // 4. 等待上链确认（设置超时）
    return withTimeout(120_000) { // 2分钟超时
        var receipt: TransactionReceipt? = null
        while (receipt == null) {
            delay(3000) // 每3秒查一次
            receipt = web3j.ethGetTransactionReceipt(txHash).send().transactionReceipt.orElse(null)
        }
        // 5. 判断成功/失败
        if (receipt.isStatusOK) MintResult.Success(txHash)
        else MintResult.Failed("合约执行失败: ${receipt.revertReason}")
    }
}
```

---

## 六、Flutter

**Q22** 🟢 Flutter 的 Widget、Element、RenderObject 三棵树是什么关系？

> **答题要点：**
> - Widget 树：不可变的配置描述（轻量，频繁重建）
> - Element 树：Widget 实例化后的状态持有者，负责 diff 对比
> - RenderObject 树：真正执行布局和绘制
> - setState 触发：Widget 重建 → Element diff → 只更新变化的 RenderObject
> - `StatefulWidget`：Widget 本身无状态，State 挂在 Element 上，所以 Widget 重建 State 不丢失

```dart
// Widget 是不可变的配置（每次 setState 都重新创建）
class CounterWidget extends StatefulWidget {
  @override
  State<CounterWidget> createState() => _CounterState();
}

// State 挂在 Element 上，Widget 重建时 State 不丢失
class _CounterState extends State<CounterWidget> {
  int _count = 0; // 保存在 State 中，不随 Widget 重建丢失

  @override
  Widget build(BuildContext context) {
    // 每次 setState 这里重新执行，但只有变化的部分更新 RenderObject
    return Column(
      children: [
        Text('$_count'),           // 只有这个 RenderObject 需要重绘
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: Text('点击'),     // 没变化，RenderObject 不重绘
        ),
      ],
    );
  }
}

/*
三棵树关系：
Widget树(配置)     Element树(实例/diff)    RenderObject树(布局绘制)
  Column      →    ColumnElement       →    RenderFlex
  Text        →    TextElement         →    RenderParagraph
  Button      →    ButtonElement       →    RenderBox
*/
```

---

**Q23** 🟡 Flutter 的状态管理你用过哪些？GetX 和 Provider 的区别？

> **答题要点：**
> - Provider：InheritedWidget 的封装，官方推荐，需要 context，适合中小项目
> - GetX：不需要 context，路由/状态/依赖注入一体化，但侵入性强
> - MobX：响应式，通过代码生成（build_runner），细粒度更新
> - 项目中混用：GetX 全局路由+状态，MobX 处理复杂页面局部状态，Provider 做依赖注入
> - 选型建议：团队小/快速开发用 GetX；长期维护项目用 Riverpod/Bloc

```dart
// Provider 方式（需要 context）
class CounterProvider extends ChangeNotifier {
  int count = 0;
  void increment() { count++; notifyListeners(); }
}

// 使用时需要 context
Widget build(BuildContext context) {
  final counter = context.watch<CounterProvider>(); // 需要 context
  return Text('${counter.count}');
}

// GetX 方式（不需要 context，更简洁）
class CounterController extends GetxController {
  var count = 0.obs; // 响应式变量
  void increment() => count++;
}

// 使用（无 context）
final controller = Get.find<CounterController>();
Obx(() => Text('${controller.count}')); // 自动监听更新

// GetX 路由（无 context）
Get.toNamed('/detail', arguments: {'id': 123});
Get.back();
```

---

**Q24** 🔴 Flutter 和原生如何通信？MethodChannel 和 EventChannel 分别用在什么场景？

> **答题要点：**
> - `MethodChannel`：双向单次调用，Flutter 调原生方法（或反向），一请求一响应
> - `EventChannel`：原生向 Flutter 持续推送事件流（如传感器、蓝牙数据）
> - `BasicMessageChannel`：传递简单消息（字符串/二进制）
> - 场景：打开相机/蓝牙 → MethodChannel；GPS 位置持续上报 → EventChannel
> - 注意：Channel 通信在平台线程（UI线程）执行，耗时操作需要切换线程

```dart
// Flutter 侧 MethodChannel（单次调用）
final channel = MethodChannel('com.yls/ble');

// Flutter 调原生：打开蓝牙扫描
Future<List<String>> scanBleDevices() async {
  final devices = await channel.invokeMethod<List>('startScan');
  return devices?.cast<String>() ?? [];
}

// Flutter 侧 EventChannel（持续接收原生推送）
final heartRateChannel = EventChannel('com.yls/heart_rate');

Stream<int> get heartRateStream {
  return heartRateChannel.receiveBroadcastStream()
      .map((event) => event as int);
}
// 使用
heartRateStream.listen((bpm) => updateUI(bpm));
```

```kotlin
// Android 原生侧实现 MethodChannel
class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        // MethodChannel：处理单次调用
        MethodChannel(flutterEngine.dartExecutor, "com.yls/ble")
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "startScan" -> {
                        val devices = bleManager.startScan()
                        result.success(devices)
                    }
                    else -> result.notImplemented()
                }
            }

        // EventChannel：持续推送心率数据
        EventChannel(flutterEngine.dartExecutor, "com.yls/heart_rate")
            .setStreamHandler(object : EventChannel.StreamHandler {
                override fun onListen(args: Any?, sink: EventChannel.EventSink) {
                    bleManager.setHeartRateCallback { bpm ->
                        sink.success(bpm) // 持续推送
                    }
                }
                override fun onCancel(args: Any?) {
                    bleManager.removeHeartRateCallback()
                }
            })
    }
}
```

---

## 七、设计模式

**Q25** 🟢 单例模式的几种实现方式？双重检查锁为什么要加 volatile？

> **答题要点：**
> - 饿汉式：类加载即初始化，线程安全，但可能浪费内存
> - 懒汉式 + synchronized：线程安全，但每次获取都加锁，性能差
> - 双重检查锁（DCL）：两次 null 检查 + synchronized 块内初始化
> - `volatile` 原因：禁止指令重排序，`new Object()` 分三步（分配内存→初始化→赋值），JIT可能重排为先赋值再初始化，另一线程拿到未初始化对象
> - Kotlin 推荐：`object` 关键字（底层是静态内部类实现，最安全简洁）

```kotlin
// ❌ 懒汉式（非线程安全）
class Singleton1 {
    companion object {
        var instance: Singleton1? = null
        fun get() = instance ?: Singleton1().also { instance = it } // 多线程会创建多个
    }
}

// ✅ 双重检查锁（Java 方式）
class Singleton2 {
    companion object {
        @Volatile  // 禁止指令重排：防止拿到未初始化的对象
        private var instance: Singleton2? = null

        fun get(): Singleton2 {
            if (instance == null) {            // 第一次检查（避免每次加锁）
                synchronized(Singleton2::class) {
                    if (instance == null) {    // 第二次检查（防止重复创建）
                        instance = Singleton2()
                    }
                }
            }
            return instance!!
        }
    }
}

// ✅ Kotlin 推荐方式：object（编译器保证线程安全）
object DatabaseManager {
    val db by lazy { Room.databaseBuilder(...).build() }
    fun query() { ... }
}

// ✅ 带参数的单例（by lazy 线程安全）
class AppDatabase private constructor(context: Context) {
    companion object {
        @Volatile private var instance: AppDatabase? = null
        fun getInstance(context: Context) = instance ?: synchronized(this) {
            instance ?: AppDatabase(context.applicationContext).also { instance = it }
        }
    }
}
```

---

**Q26** 🟡 观察者模式在 Android 开发中有哪些应用？

> **答题要点：**
> - LiveData/StateFlow：ViewModel 是被观察者，UI 是观察者
> - RxJava：Observable → Observer
> - EventBus：发布/订阅（解耦但难追踪）
> - BroadcastReceiver：系统广播
> - RecyclerView.AdapterDataObserver：Adapter 数据变化通知
> - 项目中：蓝牙数据回调 → emit 到 SharedFlow → UI 订阅更新

```kotlin
// 方式1：StateFlow（推荐，协程友好）
class HealthViewModel : ViewModel() {
    private val _bpm = MutableStateFlow(0)
    val bpm = _bpm.asStateFlow() // 被观察者

    fun updateBpm(value: Int) { _bpm.value = value }
}
// 观察者
viewModel.bpm.collect { bpm -> textView.text = "$bpm bpm" }

// 方式2：自定义观察者模式（理解原理）
interface HeartRateObserver {
    fun onHeartRateChanged(bpm: Int)
}

class BleDataSource {
    private val observers = mutableListOf<HeartRateObserver>()

    fun addObserver(o: HeartRateObserver) { observers.add(o) }
    fun removeObserver(o: HeartRateObserver) { observers.remove(o) }

    // 数据变化时通知所有观察者
    private fun notifyObservers(bpm: Int) {
        observers.forEach { it.onHeartRateChanged(bpm) }
    }
}

// 方式3：RecyclerView Adapter 通知（内置观察者）
val adapter = HealthAdapter()
adapter.submitList(newList)  // DiffUtil 计算差异，精确通知 Item 变化
```

---

**Q27** 🔴 你在项目中如何应用策略模式？举一个实际例子。

> **答题要点：**
> - 策略模式：定义算法族，分别封装，可互换，让算法独立于使用者变化
> - Android 实际场景举例：
>   - 图表渲染策略：根据数据类型选择柱状图/折线图/散点图渲染策略
>   - 支付策略：微信支付/支付宝支付/内购统一 PayStrategy 接口
>   - 网络重试策略：固定间隔/指数退避/不重试
> - 答题加分：结合简历中 AAChartView 多图表切换场景描述

```kotlin
// 结合简历：健康数据图表渲染策略
interface ChartStrategy {
    fun render(container: ViewGroup, data: List<HealthData>)
}

class BarChartStrategy : ChartStrategy {
    override fun render(container: ViewGroup, data: List<HealthData>) {
        // 用 AAChartView 渲染柱状图（步数对比）
        val chart = AAChartView(container.context)
        val model = AAChartModel().chartType(AAChartType.Column)
            .series(arrayOf(AASeriesElement().data(data.map { it.steps }.toTypedArray())))
        chart.aa_drawChartWithChartModel(model)
        container.addView(chart)
    }
}

class LineChartStrategy : ChartStrategy {
    override fun render(container: ViewGroup, data: List<HealthData>) {
        // 渲染折线图（心率趋势）
        val chart = AAChartView(container.context)
        val model = AAChartModel().chartType(AAChartType.Line)
            .series(arrayOf(AASeriesElement().data(data.map { it.bpm }.toTypedArray())))
        chart.aa_drawChartWithChartModel(model)
        container.addView(chart)
    }
}

// 使用策略（上下文）
class HealthChartView(context: Context) : FrameLayout(context) {
    private var strategy: ChartStrategy = BarChartStrategy() // 默认柱状图

    fun setChartType(type: ChartType) {
        strategy = when (type) {
            ChartType.BAR -> BarChartStrategy()
            ChartType.LINE -> LineChartStrategy()
            ChartType.SCATTER -> ScatterChartStrategy()
        }
    }

    fun show(data: List<HealthData>) {
        removeAllViews()
        strategy.render(this, data) // 切换策略，无需 if/else
    }
}
```

---

## 八、WebView / JsBridge

**Q28** 🟢 WebView 和 H5 双向通信的方式有哪些？

> **答题要点：**
> - 原生调 JS：`webView.evaluateJavascript("js代码", callback)`（异步，推荐）
> - JS 调原生：`@JavascriptInterface` 注解方法，JS 通过 `window.Android.method()` 调用
> - JsBridge：基于 iframe + URL scheme 拦截，或自定义协议，解决双向通信和类型安全问题
> - 安全注意：`@JavascriptInterface` 暴露的方法所有 JS 都能调，需要做来源校验和参数校验

```kotlin
// 原生 → JS：evaluateJavascript（异步，API 19+）
webView.evaluateJavascript("javascript:updateUserInfo('${userJson}')") { result ->
    // result 是 JS 函数的返回值（JSON 字符串）
    Log.d("WebView", "JS 返回：$result")
}

// JS → 原生：@JavascriptInterface
class AndroidBridge(private val activity: Activity) {

    @JavascriptInterface
    fun showToast(message: String) {
        // ⚠️ 此方法在子线程调用，需切回主线程操作 UI
        activity.runOnUiThread {
            Toast.makeText(activity, message, Toast.LENGTH_SHORT).show()
        }
    }

    @JavascriptInterface
    fun getUserToken(): String {
        // ⚠️ 安全：只返回必要数据，不暴露敏感信息
        return TokenStore.getToken()
    }
}

// 注册 Bridge
webView.settings.javaScriptEnabled = true
webView.addJavascriptInterface(AndroidBridge(this), "Android")

// H5 侧调用：
// window.Android.showToast("Hello from H5!")
// const token = window.Android.getUserToken()
```

---

**Q29** 🟡 WebView 加载 H5 白屏的原因有哪些？如何优化首屏加载速度？

> **答题要点：**
> - 白屏原因：网络慢、JS 包体积大、未开启硬件加速、WebView 初始化慢
> - 优化方案：
>   1. WebView 预创建：Application 中提前初始化 WebView 放入池中
>   2. 资源预加载：提前下载 H5 资源包，`WebViewClient.shouldInterceptRequest` 拦截替换
>   3. 开启缓存：`LOAD_CACHE_ELSE_NETWORK`
>   4. 开启硬件加速：`setLayerType(LAYER_TYPE_HARDWARE)`
>   5. 骨架屏：H5 先渲染骨架，数据异步填充

```kotlin
// 优化1：预创建 WebView（Application 中）
object WebViewPool {
    private var cachedWebView: WebView? = null

    fun init(context: Context) {
        // 在 Application.onCreate() 提前初始化（WebView 首次创建耗时 200-500ms）
        cachedWebView = WebView(context.applicationContext)
        cachedWebView?.loadUrl("about:blank") // 预热 WebView 进程
    }

    fun acquire(context: Context): WebView {
        return cachedWebView?.also { cachedWebView = null }
            ?: WebView(context)
    }
}

// 优化2：拦截本地资源（离线包方案）
webView.webViewClient = object : WebViewClient() {
    override fun shouldInterceptRequest(view: WebView, request: WebResourceRequest): WebResourceResponse? {
        val url = request.url.toString()
        // 如果本地有缓存资源，直接返回，不走网络
        val localFile = LocalResourceCache.get(url)
        return if (localFile != null) {
            WebResourceResponse("text/javascript", "UTF-8", localFile.inputStream())
        } else {
            super.shouldInterceptRequest(view, request)
        }
    }
}

// 优化3：WebView 基础配置
webView.settings.apply {
    javaScriptEnabled = true
    domStorageEnabled = true
    cacheMode = WebSettings.LOAD_CACHE_ELSE_NETWORK // 优先缓存
    mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
}
webView.setLayerType(View.LAYER_TYPE_HARDWARE, null) // 硬件加速
```

---

**Q30** 🔴 WebView 的内存泄漏如何避免？

> **答题要点：**
> - 原因：WebView 内部持有 Activity Context，如果 Activity 销毁时未正确释放
> - 解法1：WebView 使用 `applicationContext`（但会影响某些功能）
> - 解法2：Activity `onDestroy` 中先移除 WebView 的父 View，再调用 `webView.destroy()`
> - 解法3：将 WebView 放在独立进程，彻底隔离（`android:process=":web"`），进程可以被直接杀掉
> - 解法4：使用 X5 内核（腾讯）或 自定义 WebView 进程

```kotlin
// ❌ 错误：直接在 XML 中声明 WebView（持有 Activity Context，容易泄漏）
// <WebView android:id="@+id/webView" ... />

// ✅ 正确：代码创建，使用 ApplicationContext
class WebActivity : AppCompatActivity() {
    private var webView: WebView? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 动态创建，传 applicationContext 而非 Activity
        webView = WebView(applicationContext).also { wv ->
            wv.settings.javaScriptEnabled = true
            binding.webContainer.addView(wv)
        }
        webView?.loadUrl(url)
    }

    override fun onDestroy() {
        // 正确销毁顺序：
        webView?.let { wv ->
            // 1. 先从父 View 移除（断开 View 树引用）
            (wv.parent as? ViewGroup)?.removeView(wv)
            // 2. 停止加载、清空历史
            wv.stopLoading()
            wv.clearHistory()
            // 3. 销毁 WebView
            wv.destroy()
        }
        webView = null
        super.onDestroy()
    }
}

// ✅ 最彻底方案：独立进程（AndroidManifest.xml）
// <activity android:name=".WebActivity" android:process=":web_process" />
// Activity finish 时，:web_process 进程可以直接 kill，不影响主进程
```

---

## 九、综合 / 架构设计

**Q31** 🔴 如果让你设计 YLS 戒指 App 的蓝牙数据处理架构，你会怎么设计？

> **答题要点（考察系统设计能力）：**
> - 连接层：`BluetoothService`（前台 Service），维持 BLE 连接，处理重连
> - 数据层：解析蓝牙原始字节 → 转成领域模型（HeartRate、StepCount、SleepData）
> - 仓库层：`HealthRepository` 同时写入 Room（本地缓存）和上报服务器
> - 数据流：`MutableSharedFlow` 在 Service 内广播实时数据
> - UI 层：通过 `HealthViewModel` 订阅 Flow，聚合多指标展示
> - 解耦：Service 和 ViewModel 不直接通信，通过 Repository 单一数据源

```
整体架构图：

┌─────────────────────────────────────────────┐
│                  UI Layer                    │
│  HealthFragment  ←  HealthViewModel          │
│                      ↑ StateFlow             │
├─────────────────────────────────────────────┤
│               Domain Layer                   │
│  HealthRepository（单一数据源）               │
│       ↑ Flow          ↑ suspend              │
├──────────────┬──────────────────────────────┤
│  Local(Room) │       Remote(Retrofit)        │
├──────────────┴──────────────────────────────┤
│              BLE Service Layer               │
│  BluetoothForegroundService                  │
│    ↓ 解析字节    ↓ emit SharedFlow           │
│  RingDataParser → MutableSharedFlow<Health>  │
│    ↑ 原始数据                                │
│  BluetoothGattCallback（硬件回调）            │
└─────────────────────────────────────────────┘
```

```kotlin
// BLE Service：维持连接 + 广播数据
class BluetoothForegroundService : Service() {
    // 单例数据流（App 内共享）
    companion object {
        val healthFlow = MutableSharedFlow<HealthData>(replay = 1)
    }

    private val gattCallback = object : BluetoothGattCallback() {
        override fun onCharacteristicChanged(gatt: BluetoothGatt, char: BluetoothGattCharacteristic) {
            val health = RingDataParser.parse(char.value) // 解析字节
            serviceScope.launch { healthFlow.emit(health) }
        }
    }
}

// Repository：聚合数据来源
class HealthRepository(private val dao: HealthDao, private val api: HealthApi) {
    // 实时数据（来自 BLE）
    val realtimeHealth: SharedFlow<HealthData> = BluetoothForegroundService.healthFlow

    // 持久化 + 上报
    suspend fun save(data: HealthData) {
        dao.insert(data)                    // 存本地
        runCatching { api.sync(data) }      // 上报服务器（失败不影响本地）
    }
}

// ViewModel：消费数据
class HealthViewModel(private val repo: HealthRepository) : ViewModel() {
    val heartRate = repo.realtimeHealth
        .map { it.bpm }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 0)

    init {
        // 自动持久化
        viewModelScope.launch {
            repo.realtimeHealth.collect { repo.save(it) }
        }
    }
}
```

---

**Q32** 🔴 你引入 Claude Code 辅助开发的实际效果是什么？有没有踩过坑？

> **答题要点（考察 AI 工具应用能力）：**
> - 效果：生成 ViewModel 样板代码、单元测试、健康数据处理算法
> - CLAUDE.md：定义架构约定，让 AI 生成的代码自动对齐项目规范
> - MCP+Figma：将设计稿直接转为 Compose/XML 布局，减少还原误差
> - 踩坑1：AI 生成代码需要 Review，尤其是安全相关（密钥处理/权限申请）
> - 踩坑2：AI 对项目特定业务逻辑理解有限，需要补充上下文
> - 踩坑3：过度依赖 AI 可能导致对底层原理理解不深，面试时反而答不出来

```markdown
// CLAUDE.md 示例（定义项目规范，让 AI 生成符合规范的代码）

## 架构约定
- 使用 MVVM + Repository 模式
- ViewModel 只持有 StateFlow/SharedFlow，禁止持有 View 引用
- Repository 是唯一数据来源，禁止 ViewModel 直接访问 Dao 或 Api

## 代码规范
- 协程：UI 订阅必须使用 repeatOnLifecycle(STARTED)
- 错误处理：统一在 Repository 层 try-catch，ViewModel 只处理业务逻辑
- 命名：Flow 变量名以 Flow 结尾，StateFlow 以 State 结尾

## 禁止事项
- 禁止在 ViewModel 中直接 import android.* View 相关类
- 禁止使用 GlobalScope
- 禁止硬编码 API Key
```

---

## 附：面试高频追问

| 说了这个 | 面试官可能追问 |
|----------|---------------|
| 用了协程 | 协程是如何实现挂起的？CPS 变换 |
| 用了 Flow | Flow 的背压是如何处理的？`buffer()`/`conflate()` |
| 用了 MVVM | 为什么不用 MVI？MVI 的单向数据流优势 |
| 做了性能优化 | 优化前后有没有量化数据？工具怎么用的 |
| 用了 BLE | 你们戒指的协议是自定义的还是标准的？解析协议遇到过什么问题 |
| 做了 Web3 | 私钥在哪存储？如何防止助记词泄露 |
| 用了 Flutter | Flutter 为什么比 RN 性能好？Dart VM 和 AOT 的区别 |

---

*生成时间：2026-03-10 | 共 32 题 + 代码示例 + 追问清单*
