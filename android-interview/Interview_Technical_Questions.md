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

---

**Q2** 🟡 ViewModel 为什么能在 Activity 旋转后保留数据？

> **答题要点：**
> - ViewModel 由 `ViewModelStore` 持有，Activity 旋转时系统重建 Activity 但保留 ViewModelStore
> - ViewModel 生命周期绑定到 ViewModelStoreOwner（Activity/Fragment）的生命周期，直到 `onCleared()` 真正销毁
> - 注意：进程被杀后 ViewModel 不保留，需配合 `SavedStateHandle`

---

**Q3** 🔴 LiveData 和 StateFlow 的区别？你在项目里怎么选择？

> **答题要点：**
> - LiveData：生命周期感知，自动取消订阅，粘性事件（新观察者会收到最新值）
> - StateFlow：需手动处理生命周期（`lifecycleScope.repeatOnLifecycle`），初始值必须有，适合状态
> - SharedFlow：无初始值，适合一次性事件（Toast、导航）
> - 选择原则：UI 状态 → StateFlow；一次性事件 → SharedFlow；纯 UI 更新且不需要协程 → LiveData
> - 项目中：YLS 健康数据用 StateFlow，网络错误提示用 SharedFlow

---

### 2. Kotlin 协程 & Flow

**Q4** 🟢 协程和线程的区别？协程是轻量级的原因是什么？

> **答题要点：**
> - 线程：OS 级别，切换需要内核参与，栈空间固定（默认 1MB+）
> - 协程：用户态调度，挂起/恢复不需要 OS 介入，挂起时只保存极少状态
> - 轻量：协程挂起时不阻塞线程，一个线程可以运行成千上万协程
> - 本质：编译器将 suspend 函数转成状态机（CPS 变换）

---

**Q5** 🟡 `Flow` 的冷流和热流有什么区别？举例说明。

> **答题要点：**
> - 冷流（Cold）：每次 collect 都会重新执行，`flow { }` 构建的是冷流
> - 热流（Hot）：不管有没有订阅者都在运行，`StateFlow`、`SharedFlow` 是热流
> - 场景：网络请求用冷流（按需触发）；传感器数据、蓝牙数据用热流（持续推送）
> - YLS 中蓝牙心率数据：Ring SDK 回调 → `MutableSharedFlow.emit()` → UI 订阅

---

**Q6** 🔴 `launchIn`、`lifecycleScope.launch`、`repeatOnLifecycle` 的区别和使用场景？

> **答题要点：**
> - `lifecycleScope.launch`：在 DESTROYED 才取消，页面不可见时仍然收集，浪费资源
> - `launchIn(lifecycleScope)`：同上，只是写法不同
> - `repeatOnLifecycle(Lifecycle.State.STARTED)`：在 STARTED 时收集，STOPPED 时自动取消，页面回到前台重新订阅
> - **结论**：UI 相关的 Flow 订阅必须用 `repeatOnLifecycle`，否则在后台也会更新 UI 可能崩溃

---

### 3. Jetpack 组件

**Q7** 🟡 Room 数据库升级时如何处理迁移？如果字段变更怎么办？

> **答题要点：**
> - 提供 `Migration(oldVersion, newVersion)` 编写 SQL
> - `fallbackToDestructiveMigration()` 开发期可用，生产环境不能用（数据丢失）
> - 字段新增：`ALTER TABLE ... ADD COLUMN`
> - 字段删除/修改：创建新表 → 复制数据 → 删旧表 → 重命名
> - 建议：结合 `@Database(exportSchema = true)` 导出 schema 文件做版本追踪

---

## 二、蓝牙 / BLE 开发

**Q8** 🟢 BLE 和经典蓝牙的区别？戒指类设备为什么用 BLE？

> **答题要点：**
> - 经典蓝牙：高带宽（音频/文件传输），功耗高
> - BLE：低功耗（可用纽扣电池运行数月），适合小数据量周期上报
> - 戒指原因：电池极小，健康数据（心率、步数）数据量小，BLE 完全够用

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

---

**Q11** 🔴 BLE 的 MTU 是什么？如何传输大数据包？

> **答题要点：**
> - MTU（Maximum Transmission Unit）：单次传输最大字节数，默认 23 字节（实际数据 20 字节）
> - 协商：`gatt.requestMtu(512)` → `onMtuChanged` 回调实际值
> - 大数据传输（如固件升级 OTA）：手动分包，每包不超过 MTU-3 字节
> - 注意：部分设备不支持大 MTU，需要做降级处理

---

## 三、性能优化

**Q12** 🟢 ANR 的触发条件是什么？如何定位和排查？

> **答题要点：**
> - 触发条件：主线程 5s 内无响应（按键/触摸）；BroadcastReceiver 前台10s/后台60s；Service 前台20s/后台200s
> - 排查：`/data/anr/traces.txt`，查看主线程堆栈
> - 常见原因：主线程执行 IO/网络/锁等待、死锁、Binder 调用耗时
> - 线上监控：`BlockCanary`、`ANR-WatchDog`、`Matrix`（腾讯）
> - 预防：`StrictMode` 开发期检测主线程 IO

---

**Q13** 🟡 OOM 常见场景和解决方案？

> **答题要点：**
> - 场景1：Bitmap 未压缩直接加载大图 → 使用 Glide/Coil，设置 `override()` 尺寸
> - 场景2：内存泄漏积累（Activity 被静态持有）→ LeakCanary 检测
> - 场景3：大量对象频繁创建触发 GC → 对象池复用
> - 场景4：WebView 导致内存泄漏 → 在独立进程中运行 WebView
> - 分析工具：Android Profiler Memory、MAT（Memory Analyzer Tool）
> - Bitmap 优化：`inSampleSize` 降采样、`RGB_565` 替代 `ARGB_8888`（内存减半）

---

**Q14** 🔴 Android 15 的 16KB Page Size 适配，你做了哪些工作？

> **答题要点：**
> - 背景：Android 15 起支持 16KB 内存页（原来 4KB），提升性能但不兼容旧 .so
> - 需要适配的：所有 native .so 库必须按 16KB 对齐编译
> - 检测方法：`adb shell /system/bin/linker_config_checker` 或 Android Studio 提示
> - 编译修改：CMakeLists.txt 中 `target_link_options(... -Wl,-z,max-page-size=16384)`
> - 三方库：需要等待三方库更新，或自行重编译
> - 验证：在 Android 15 模拟器（16KB模式）运行测试

---

**Q15** 🟡 Glide 的缓存机制是怎样的？

> **答题要点：**
> - 四级缓存：活跃缓存（弱引用，当前显示）→ 内存缓存（LruCache）→ 磁盘缓存 → 网络
> - 磁盘缓存两种：`DATA`（原始数据）和 `RESOURCE`（解码后的 Bitmap）
> - 缓存 Key：由 URL + 宽高 + 变换等组合生成
> - 跳过缓存：`.skipMemoryCache(true).diskCacheStrategy(DiskCacheStrategy.NONE)`

---

## 四、网络层

**Q16** 🟢 OkHttp 的拦截器链是怎么工作的？

> **答题要点：**
> - 责任链模式：`Chain.proceed()` 将请求传递给下一个拦截器
> - 顺序（从外到内）：自定义拦截器 → RetryAndFollowUp → Bridge → Cache → Connect → Network
> - 应用拦截器（`addInterceptor`）：不管缓存，只调用一次，适合日志/全局Header
> - 网络拦截器（`addNetworkInterceptor`）：可看到实际网络请求，重定向时会多次调用
> - 项目实践：添加 Token 拦截器、日志拦截器、错误统一处理拦截器

---

**Q17** 🟡 Retrofit 是如何通过注解生成接口实现的？

> **答题要点：**
> - 动态代理：`Retrofit.create(ApiService::class.java)` 返回代理对象
> - 调用方法时，`InvocationHandler` 拦截，解析方法上的注解（@GET/@POST/@Path等）
> - 构建 `ServiceMethod`（含 OkHttp Request 构建逻辑）
> - 通过 `CallAdapter` 转换返回类型（Call → Deferred → Flow）
> - 通过 `Converter` 转换请求/响应体（Gson/Moshi）

---

**Q18** 🔴 如何实现 Token 过期自动刷新，且多个并发请求只刷新一次？

> **答题要点：**
> - 在 OkHttp 拦截器中处理 401 响应
> - 多并发问题：用 `Mutex`（协程）或 `synchronized` 加锁
> - 思路：第一个线程触发 refresh，其余线程挂起等待；refresh 完成后所有线程用新 token 重试
> - 代码关键：`@Volatile var isRefreshing` + 双重检查，或协程 `Mutex.withLock { }`
> - 注意：refresh 本身的请求不能再经过这个拦截器（避免死循环）

---

## 五、Web3 / 区块链

**Q19** 🟢 你在项目中接入 WalletConnect 的流程是怎样的？

> **答题要点：**
> - WalletConnect v2 基于 WebSocket 中继服务器，App 和钱包通过 topic 通信
> - 流程：App 生成 URI → 用户用钱包扫码 → 建立会话（Session）→ 发起请求（签名/交易）→ 钱包确认
> - Android SDK：`WalletConnect` 官方 Kotlin SDK
> - 注意：URI 有时效性，session 过期需要重新连接

---

**Q20** 🟡 web3j 调用智能合约的方式有哪些？load 和 deploy 的区别？

> **答题要点：**
> - `Contract.deploy()`：部署新合约到链上，返回合约地址
> - `Contract.load(address, ...)`：加载已部署合约，通过地址交互
> - 调用只读方法（`view/pure`）：`call()`，不消耗 Gas，不上链
> - 调用写入方法：`send()`，需要签名和 Gas，等待 TransactionReceipt
> - ABI：合约的接口描述文件，web3j 通过 ABI 生成 Java/Kotlin Wrapper 类

---

**Q21** 🔴 NFT 铸造的链上流程是什么？你是如何处理交易失败和 Gas 估算的？

> **答题要点：**
> - 铸造：调用合约 `mint(to, tokenId)` 方法，写入链上
> - Gas 估算：`web3j.ethEstimateGas(transaction).send()`，实际设置时乘以 1.2 留余量
> - Gas Price：`ethGasPrice` 获取当前网络建议价格，或用 EIP-1559（maxFeePerGas）
> - 失败处理：监听 TransactionReceipt 的 `status`（"0x1" 成功，"0x0" 失败）
> - 超时处理：设置超时，超时后查询 txHash 状态，而不是无限等待
> - 用户体验：铸造过程中显示进度，Hash 生成后即可给用户看（乐观更新）

---

## 六、Flutter

**Q22** 🟢 Flutter 的 Widget、Element、RenderObject 三棵树是什么关系？

> **答题要点：**
> - Widget 树：不可变的配置描述（轻量，频繁重建）
> - Element 树：Widget 实例化后的状态持有者，负责 diff 对比
> - RenderObject 树：真正执行布局和绘制
> - setState 触发：Widget 重建 → Element diff → 只更新变化的 RenderObject
> - `StatefulWidget`：Widget 本身无状态，State 挂在 Element 上，所以 Widget 重建 State 不丢失

---

**Q23** 🟡 Flutter 的状态管理你用过哪些？GetX 和 Provider 的区别？

> **答题要点：**
> - Provider：InheritedWidget 的封装，官方推荐，需要 context，适合中小项目
> - GetX：不需要 context，路由/状态/依赖注入一体化，但侵入性强
> - MobX：响应式，通过代码生成（build_runner），细粒度更新
> - 项目中混用：GetX 全局路由+状态，MobX 处理复杂页面局部状态，Provider 做依赖注入
> - 选型建议：团队小/快速开发用 GetX；长期维护项目用 Riverpod/Bloc

---

**Q24** 🔴 Flutter 和原生如何通信？MethodChannel 和 EventChannel 分别用在什么场景？

> **答题要点：**
> - `MethodChannel`：双向单次调用，Flutter 调原生方法（或反向），一请求一响应
> - `EventChannel`：原生向 Flutter 持续推送事件流（如传感器、蓝牙数据）
> - `BasicMessageChannel`：传递简单消息（字符串/二进制）
> - 场景：打开相机/蓝牙 → MethodChannel；GPS 位置持续上报 → EventChannel
> - 注意：Channel 通信在平台线程（UI线程）执行，耗时操作需要切换线程

---

## 七、设计模式

**Q25** 🟢 单例模式的几种实现方式？双重检查锁为什么要加 volatile？

> **答题要点：**
> - 饿汉式：类加载即初始化，线程安全，但可能浪费内存
> - 懒汉式 + synchronized：线程安全，但每次获取都加锁，性能差
> - 双重检查锁（DCL）：两次 null 检查 + synchronized 块内初始化
> - `volatile` 原因：禁止指令重排序，`new Object()` 分三步（分配内存→初始化→赋值），JIT可能重排为先赋值再初始化，另一线程拿到未初始化对象
> - Kotlin 推荐：`object` 关键字（底层是静态内部类实现，最安全简洁）

---

**Q26** 🟡 观察者模式在 Android 开发中有哪些应用？

> **答题要点：**
> - LiveData/StateFlow：ViewModel 是被观察者，UI 是观察者
> - RxJava：Observable → Observer
> - EventBus：发布/订阅（解耦但难追踪）
> - BroadcastReceiver：系统广播
> - RecyclerView.AdapterDataObserver：Adapter 数据变化通知
> - 项目中：蓝牙数据回调 → emit 到 SharedFlow → UI 订阅更新

---

**Q27** 🔴 你在项目中如何应用策略模式？举一个实际例子。

> **答题要点：**
> - 策略模式：定义算法族，分别封装，可互换，让算法独立于使用者变化
> - Android 实际场景举例：
>   - 图表渲染策略：根据数据类型选择柱状图/折线图/散点图渲染策略
>   - 支付策略：微信支付/支付宝支付/内购统一 PayStrategy 接口
>   - 网络重试策略：固定间隔/指数退避/不重试
> - 答题加分：结合简历中 AAChartView 多图表切换场景描述

---

## 八、WebView / JsBridge

**Q28** 🟢 WebView 和 H5 双向通信的方式有哪些？

> **答题要点：**
> - 原生调 JS：`webView.evaluateJavascript("js代码", callback)`（异步，推荐）
> - JS 调原生：`@JavascriptInterface` 注解方法，JS 通过 `window.Android.method()` 调用
> - JsBridge：基于 iframe + URL scheme 拦截，或自定义协议，解决双向通信和类型安全问题
> - 安全注意：`@JavascriptInterface` 暴露的方法所有 JS 都能调，需要做来源校验和参数校验

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

---

**Q30** 🔴 WebView 的内存泄漏如何避免？

> **答题要点：**
> - 原因：WebView 内部持有 Activity Context，如果 Activity 销毁时未正确释放
> - 解法1：WebView 使用 `applicationContext`（但会影响某些功能）
> - 解法2：Activity `onDestroy` 中先移除 WebView 的父 View，再调用 `webView.destroy()`
> - 解法3：将 WebView 放在独立进程，彻底隔离（`android:process=":web"`），进程可以被直接杀掉
> - 解法4：使用 X5 内核（腾讯）或 自定义 WebView 进程

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

---

**Q32** 🔴 你引入 Claude Code 辅助开发的实际效果是什么？有没有踩过坑？

> **答题要点（考察 AI 工具应用能力）：**
> - 效果：生成 ViewModel 样板代码、单元测试、健康数据处理算法
> - CLAUDE.md：定义架构约定，让 AI 生成的代码自动对齐项目规范
> - MCP+Figma：将设计稿直接转为 Compose/XML 布局，减少还原误差
> - 踩坑1：AI 生成代码需要 Review，尤其是安全相关（密钥处理/权限申请）
> - 踩坑2：AI 对项目特定业务逻辑理解有限，需要补充上下文
> - 踩坑3：过度依赖 AI 可能导致对底层原理理解不深，面试时反而答不出来

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

*生成时间：2026-03-10 | 共 32 题 + 追问清单*
