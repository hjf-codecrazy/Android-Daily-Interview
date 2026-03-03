# Android 完整学习笔记 - Part 1

> 包含详细答案 + 代码示例，适合深度学习和理解
>
> 共 22 题：Android 四大组件（13题）+ 异步消息机制（9题）

---

## 📌 Android 四大组件

### 1. Activity 与 Fragment 之间常见的几种通信方式

#### 详细答案

Activity 和 Fragment 之间的通信是 Android 开发中的常见场景。主要有以下几种方式：

| 方式 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **接口回调** | Fragment 向 Activity 传递 | 类型安全、代码清晰 | 需要定义接口 |
| **Bundle** | 初始化参数、简单数据 | 简单、可靠 | 只能传序列化数据 |
| **ViewModel** | 共享数据、MVVM 架构 | 生命周期感知、数据持久化 | 需要 AndroidX 库 |
| **EventBus** | 解耦通信 | 灵活、代码简洁 | 引入第三方库 |
| **LocalBroadcastManager** | 全局事件 | 安全性高 | 性能一般 |
| **直接方法调用** | 快速开发 | 简单直接 | 耦合度高、不推荐 |

#### 代码示例

**方式 1：接口回调（推荐）**

```java
// 1. 定义回调接口
public interface OnFragmentInteractionListener {
    void onFragmentMessage(String message);
}

// 2. Fragment 中持有接口引用
public class MessageFragment extends Fragment {
    private OnFragmentInteractionListener listener;

    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        // 获取 Activity 的引用并转换为接口
        if (context instanceof OnFragmentInteractionListener) {
            listener = (OnFragmentInteractionListener) context;
        }
    }

    public void sendMessageToActivity(String message) {
        if (listener != null) {
            listener.onFragmentMessage(message);
        }
    }

    @Override
    public void onDetach() {
        super.onDetach();
        listener = null;  // 防止内存泄漏
    }
}

// 3. Activity 实现接口
public class MainActivity extends AppCompatActivity implements OnFragmentInteractionListener {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    public void onFragmentMessage(String message) {
        Toast.makeText(this, "Fragment 传来: " + message, Toast.LENGTH_SHORT).show();
    }
}

// 4. Activity 向 Fragment 传递数据
public void sendDataToFragment(String data) {
    MessageFragment fragment = (MessageFragment) getSupportFragmentManager()
        .findFragmentByTag("message_fragment");
    if (fragment != null) {
        fragment.receiveData(data);
    }
}
```

**方式 2：Bundle（传递初始化参数）**

```java
// Fragment 中
public class DetailFragment extends Fragment {
    private static final String ARG_ID = "id";
    private static final String ARG_NAME = "name";

    public static DetailFragment newInstance(String id, String name) {
        DetailFragment fragment = new DetailFragment();
        Bundle args = new Bundle();
        args.putString(ARG_ID, id);
        args.putString(ARG_NAME, name);
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (getArguments() != null) {
            String id = getArguments().getString(ARG_ID);
            String name = getArguments().getString(ARG_NAME);
        }
    }
}

// Activity 中
public class MainActivity extends AppCompatActivity {
    private void loadFragment() {
        DetailFragment fragment = DetailFragment.newInstance("123", "Tom");
        getSupportFragmentManager()
            .beginTransaction()
            .replace(R.id.container, fragment)
            .addToBackStack(null)
            .commit();
    }
}
```

**方式 3：ViewModel（MVVM 架构，推荐用于复杂应用）**

```java
// 共享的 ViewModel
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<String> selectedMessage = new MutableLiveData<>();

    public void setMessage(String message) {
        selectedMessage.setValue(message);
    }

    public LiveData<String> getMessage() {
        return selectedMessage;
    }
}

// Fragment A 中发送数据
public class FragmentA extends Fragment {
    private SharedViewModel viewModel;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_a, container, false);
    }

    @Override
    public void onViewCreated(View view, Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        viewModel = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);

        Button sendBtn = view.findViewById(R.id.send_btn);
        sendBtn.setOnClickListener(v -> {
            viewModel.setMessage("Message from Fragment A");
        });
    }
}

// Fragment B 中接收数据
public class FragmentB extends Fragment {
    private SharedViewModel viewModel;
    private TextView messageView;

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_b, container, false);
    }

    @Override
    public void onViewCreated(View view, Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        viewModel = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
        messageView = view.findViewById(R.id.message_view);

        // 观察数据变化
        viewModel.getMessage().observe(getViewLifecycleOwner(), message -> {
            messageView.setText(message);
        });
    }
}
```

**方式 4：EventBus（解耦通信）**

```java
// 事件类
public class MessageEvent {
    public String message;

    public MessageEvent(String message) {
        this.message = message;
    }
}

// Fragment 中发送事件
public class SenderFragment extends Fragment {
    public void sendMessage() {
        EventBus.getDefault().post(new MessageEvent("Hello from Fragment"));
    }
}

// 另一个 Fragment 或 Activity 中接收事件
public class ReceiverFragment extends Fragment {
    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Override
    public void onStop() {
        EventBus.getDefault().unregister(this);
        super.onStop();
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessageEvent(MessageEvent event) {
        Toast.makeText(getContext(), event.message, Toast.LENGTH_SHORT).show();
    }
}
```

#### 核心要点

```
接口回调    → 类型安全，推荐用于 Activity ↔ Fragment
Bundle      → 初始化参数，推荐用于创建 Fragment 时传递数据
ViewModel   → 共享数据，推荐用于 MVVM 架构，生命周期感知
EventBus    → 全局事件，推荐用于解耦的复杂通信
直接调用    → 避免使用，耦合度太高
```

#### 选择建议

```
场景                          推荐方案
─────────────────────────────────────────
初始化参数                      Bundle
Activity ↔ Fragment             接口回调
多个 Fragment 间通信             ViewModel
全局事件通知                    EventBus
同步获取结果                    接口回调
```

---

### 2. Android 中几种 LaunchMode 的特点和应用场景

#### 详细答案

LaunchMode（启动模式）决定了 Activity 在栈中的行为方式。Android 提供了 4 种启动模式。

| LaunchMode | 栈行为 | 创建新实例 | 常见场景 |
|------------|--------|----------|---------|
| **standard** | 标准栈 | 每次都创建 | 普通 Activity（默认） |
| **singleTop** | 栈顶检查 | 栈顶有则复用 | 通知点击、Feed 列表 |
| **singleTask** | 栈内检查 | 栈内有则复用，移到栈顶 | 应用入口、浏览器 |
| **singleInstance** | 独立栈 | 单独栈中唯一实例 | 来电界面、系统应用 |

#### 代码示例

**AndroidManifest.xml 配置**

```xml
<!-- standard 模式（默认） -->
<activity android:name=".StandardActivity" />

<!-- singleTop 模式 -->
<activity
    android:name=".SingleTopActivity"
    android:launchMode="singleTop" />

<!-- singleTask 模式 -->
<activity
    android:name=".SingleTaskActivity"
    android:launchMode="singleTask" />

<!-- singleInstance 模式 -->
<activity
    android:name=".SingleInstanceActivity"
    android:launchMode="singleInstance" />
```

**程序化方式启动（通过 Intent Flag）**

```java
public class LaunchModeDemo {

    // standard 模式（每次创建新实例）
    public void startStandardActivity() {
        Intent intent = new Intent(this, StandardActivity.class);
        startActivity(intent);  // 创建新实例，可能有多个
    }

    // singleTop 模式（栈顶存在则复用）
    public void startSingleTopActivity() {
        Intent intent = new Intent(this, SingleTopActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
        startActivity(intent);
    }

    // singleTask 模式（栈内存在则复用）
    public void startSingleTaskActivity() {
        Intent intent = new Intent(this, SingleTaskActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
        startActivity(intent);
    }

    // singleInstance 模式（独立栈）
    public void startSingleInstanceActivity() {
        Intent intent = new Intent(this, SingleInstanceActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_SINGLE_TASK);
        startActivity(intent);
    }
}
```

**生命周期演示**

```java
public class SingleTopActivity extends AppCompatActivity {

    private static final String TAG = "SingleTopActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_single_top);
        Log.d(TAG, "onCreate");
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        Log.d(TAG, "onNewIntent - 栈顶复用，不调用 onCreate");
        // 这里处理新的 Intent 数据
        Bundle extras = intent.getExtras();
    }

    @Override
    protected void onStart() {
        super.onStart();
        Log.d(TAG, "onStart");
    }

    @Override
    protected void onResume() {
        super.onResume();
        Log.d(TAG, "onResume");
    }
}
```

**实际应用场景**

```java
// 场景 1：Feed 流（singleTop）- 防止重复创建
public class FeedActivity extends AppCompatActivity {
    // 用户在 Feed 页面刷新、点击通知时，都回到同一个实例
    // 避免创建多个 FeedActivity
}

// 场景 2：应用首页（singleTask）- 确保唯一实例
public class MainActivity extends AppCompatActivity {
    // 按 Home 键再回到应用，回到首页而不是其他 Activity
    // 确保只有一个实例
}

// 场景 3：浏览器首页（singleTask）
public class BrowserActivity extends AppCompatActivity {
    // 从各种应用点击链接打开浏览器
    // 都回到同一个浏览器窗口
}

// 场景 4：来电界面（singleInstance）
public class IncomingCallActivity extends AppCompatActivity {
    // 来电时显示来电界面
    // 在独立的栈中，不与其他 Activity 混合
}
```

**栈演示**

```
启动顺序：A → B → C

standard 模式：
A → [A] → [A, B] → [A, B, C]

singleTop 模式：
A → [A] → [A, B] → [A, B, C]（C 栈顶，再启动 C）→ [A, B, C]（不创建新 C）

singleTask 模式：
A → [A] → [A, B] → [A, B, C]（再启动 B）→ [A, B]（清除 C 之上的）

singleInstance 模式：
A → [A] 栈1 → [A] 栈1 + [B] 栈2 → [A] 栈1 + [B, C] 栈2
```

#### 核心要点

```
standard    → 默认，每次创建新实例，可能导致栈很深
singleTop   → 栈顶存在则复用，适合 Feed、通知场景
singleTask  → 栈内唯一，适合应用首页、浏览器
singleInstance → 独立栈，适合来电、系统应用
```

---

### 3. BroadcastReceiver 与 LocalBroadcastManager 的区别

#### 详细答案

| 特性 | BroadcastReceiver | LocalBroadcastManager |
|------|------------------|----------------------|
| **应用范围** | 全局（所有应用） | 本应用内 |
| **安全性** | 低（其他应用可拦截） | 高（隔离应用边界） |
| **性能** | 一般（跨进程） | 优异（进程内） |
| **注册方式** | 静态/动态 | 动态（仅限应用内） |
| **权限** | 需要声明权限 | 不需要权限 |
| **使用场景** | 系统事件（开机、来电） | 应用内事件通知 |

#### 代码示例

**BroadcastReceiver（全局广播）**

```java
// 1. 定义接收器
public class MyBroadcastReceiver extends BroadcastReceiver {
    private static final String TAG = "MyBroadcastReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        Log.d(TAG, "收到广播: " + action);

        if ("com.example.MY_BROADCAST".equals(action)) {
            String data = intent.getStringExtra("data");
            Toast.makeText(context, "数据: " + data, Toast.LENGTH_SHORT).show();
        }
    }
}

// 2. 在 AndroidManifest.xml 中注册（静态注册）
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="com.example.MY_BROADCAST" />
    </intent-filter>
</receiver>

// 3. 或者在 Activity 中动态注册
public class MainActivity extends AppCompatActivity {
    private MyBroadcastReceiver receiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 动态注册
        receiver = new MyBroadcastReceiver();
        IntentFilter filter = new IntentFilter();
        filter.addAction("com.example.MY_BROADCAST");
        registerReceiver(receiver, filter, Context.RECEIVER_EXPORTED);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 注销
        if (receiver != null) {
            unregisterReceiver(receiver);
        }
    }

    // 发送广播
    public void sendBroadcast() {
        Intent intent = new Intent("com.example.MY_BROADCAST");
        intent.putExtra("data", "Hello World");
        sendBroadcast(intent);
    }
}

// 4. 系统广播监听
public class SystemBroadcastReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        if (Intent.ACTION_BOOT_COMPLETED.equals(intent.getAction())) {
            // 设备启动完成
            startService(new Intent(context, MyService.class));
        } else if (Intent.ACTION_BATTERY_LOW.equals(intent.getAction())) {
            // 电量低
            Log.d("System", "电量低");
        }
    }
}
```

**LocalBroadcastManager（应用内广播，推荐）**

```java
public class LocalBroadcastDemo {

    public static class DataChangedReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            String data = intent.getStringExtra("key");
            Toast.makeText(context, "数据更新: " + data, Toast.LENGTH_SHORT).show();
        }
    }
}

public class MainActivity extends AppCompatActivity {
    private DataChangedReceiver receiver;
    private LocalBroadcastManager localBroadcastManager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        localBroadcastManager = LocalBroadcastManager.getInstance(this);

        // 注册本地广播
        receiver = new DataChangedReceiver();
        IntentFilter filter = new IntentFilter("com.example.DATA_CHANGED");
        localBroadcastManager.registerReceiver(receiver, filter);

        Button sendBtn = findViewById(R.id.send_btn);
        sendBtn.setOnClickListener(v -> sendLocalBroadcast());
    }

    private void sendLocalBroadcast() {
        Intent intent = new Intent("com.example.DATA_CHANGED");
        intent.putExtra("key", "新数据");
        // 发送本地广播
        localBroadcastManager.sendBroadcast(intent);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 注销本地广播
        if (receiver != null) {
            localBroadcastManager.unregisterReceiver(receiver);
        }
    }
}
```

**使用 LiveData 替代（现代方案）**

```java
public class DataRepository {
    private static final MutableLiveData<String> dataLiveData = new MutableLiveData<>();

    public static LiveData<String> getDataLiveData() {
        return dataLiveData;
    }

    public static void updateData(String data) {
        dataLiveData.setValue(data);
    }
}

public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 观察数据变化，自动化更新 UI
        DataRepository.getDataLiveData().observe(this, data -> {
            Toast.makeText(this, "数据更新: " + data, Toast.LENGTH_SHORT).show();
        });

        findViewById(R.id.update_btn).setOnClickListener(v -> {
            DataRepository.updateData("新数据");
        });
    }
}
```

#### 核心要点

```
BroadcastReceiver
├─ 适合：系统事件（开机、来电、电量）、应用间通信
├─ 安全性：低（任何应用可拦截）
└─ 权限：需要声明

LocalBroadcastManager
├─ 适合：应用内事件通知
├─ 安全性：高（仅本应用）
└─ 性能：更优（进程内）

现代方案：LiveData / EventBus（更推荐）
```

---

### 4. 对于 Context，你了解多少？

#### 详细答案

Context 是 Android 应用的上下文对象，提供对全局环境的访问。

**Context 的两个实现类：**

```
Context（抽象类）
├── ContextImpl（真实实现）
│   ├── Activity
│   ├── Service
│   └── Application
└── ContextWrapper（包装类）
    ├── Activity
    ├── Service
    └── Application（包装 ContextImpl）
```

| 类型 | 生命周期 | 使用场景 | 推荐度 |
|------|---------|---------|--------|
| **Application Context** | 应用生命周期 | 全局单例、长生命周期操作 | ⭐⭐⭐⭐⭐ |
| **Activity Context** | Activity 生命周期 | UI 操作、Dialog、Toast | ⭐⭐⭐⭐⭐ |
| **Service Context** | Service 生命周期 | Service 内部操作 | ⭐⭐⭐⭐ |

#### 代码示例

```java
public class ContextDemo {

    // 获取 Context 的不同方式
    public static class ContextGetter {

        // 1. 在 Activity 中获取
        public class MyActivity extends AppCompatActivity {
            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);

                // Activity Context
                Context activityContext = this;
                Context activityContext2 = MyActivity.this;

                // Application Context
                Context appContext = getApplicationContext();
                Context appContext2 = this.getApplicationContext();
            }
        }

        // 2. 在 Service 中获取
        public class MyService extends Service {
            @Override
            public int onStartCommand(Intent intent, int flags, int startId) {
                Context serviceContext = this;
                Context appContext = getApplicationContext();
                return super.onStartCommand(intent, flags, startId);
            }

            @Override
            public IBinder onBind(Intent intent) {
                return null;
            }
        }

        // 3. 在 Application 中获取
        public class MyApplication extends Application {
            @Override
            public void onCreate() {
                super.onCreate();
                Context appContext = this;
            }
        }
    }

    // Context 的用途示例
    public static class ContextUsage {

        // 1. 获取系统服务
        public void getSystemServices(Context context) {
            // 获取各种系统服务
            ActivityManager activityManager =
                (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);

            WindowManager windowManager =
                (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

            LayoutInflater inflater =
                LayoutInflater.from(context);

            NotificationManager notificationManager =
                (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);
        }

        // 2. 文件操作
        public void fileOperations(Context context) {
            // 获取应用数据目录
            File filesDir = context.getFilesDir();
            File cacheDir = context.getCacheDir();
            File databasePath = context.getDatabasePath("database.db");

            // 读写文件
            try {
                FileOutputStream fos = context.openFileOutput("data.txt", Context.MODE_PRIVATE);
                fos.write("data".getBytes());
                fos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        // 3. SharedPreferences
        public void sharedPreferences(Context context) {
            SharedPreferences sp = context.getSharedPreferences("config", Context.MODE_PRIVATE);
            sp.edit().putString("key", "value").apply();
            String value = sp.getString("key", "default");
        }

        // 4. 资源访问
        public void accessResources(Context context) {
            // 获取资源
            String appName = context.getResources().getString(R.string.app_name);
            int color = context.getResources().getColor(R.color.primary);
            Drawable drawable = context.getResources().getDrawable(R.drawable.ic_launcher);

            // 获取尺寸
            int screenWidth = context.getResources().getDisplayMetrics().widthPixels;
            int screenHeight = context.getResources().getDisplayMetrics().heightPixels;
        }

        // 5. Intent 和 Activity 操作
        public void intentOperations(Context context) {
            // 启动 Activity
            Intent intent = new Intent(context, MainActivity.class);
            context.startActivity(intent);

            // 启动 Service
            Intent serviceIntent = new Intent(context, MyService.class);
            context.startService(serviceIntent);

            // 发送广播
            Intent broadcastIntent = new Intent("com.example.ACTION");
            context.sendBroadcast(broadcastIntent);
        }

        // 6. 显示 Toast / Dialog
        public void showUI(Context context) {
            // Toast（需要 Activity Context 或 Application Context）
            Toast.makeText(context, "Hello", Toast.LENGTH_SHORT).show();

            // Dialog（必须使用 Activity Context）
            AlertDialog.Builder builder = new AlertDialog.Builder(context);
            builder.setTitle("Dialog")
                .setMessage("Content")
                .setPositiveButton("OK", null)
                .show();
        }
    }

    // Context 内存泄漏案例
    public static class MemoryLeakExample {

        // ❌ 错误：Activity Context 生命周期太短
        public static Context sContext;  // 静态引用

        public void leakActivity(Context context) {
            sContext = context;  // Activity 销毁后仍被引用，导致泄漏
        }

        // ✓ 正确：使用 Application Context
        public void correctUsage(Context context) {
            Context appContext = context.getApplicationContext();  // 生命周期与应用相同
            // 可以安全地保存到静态变量
        }

        // ✓ 正确：不保存 Context 引用
        public void anotherCorrectUsage(Context context) {
            // 在需要时获取，不保存
            String name = context.getResources().getString(R.string.app_name);
        }
    }

    // Context 的关键规则
    public static class ContextRules {

        /*
        规则 1：避免 Context 内存泄漏
        - ❌ 不要保存 Activity Context 到静态变量
        - ✓ 如果必须保存，使用 Application Context

        规则 2：Context 的创建
        - Activity/Service/Application 都继承 Context
        - 通过 getApplicationContext() 获取应用级别 Context

        规则 3：选择正确的 Context
        - UI 操作（Toast、Dialog）→ Activity Context
        - 长生命周期操作 → Application Context
        - 启动 Activity → 任何 Context

        规则 4：Context 的作用范围
        - 获取系统服务
        - 访问资源
        - 文件和数据库操作
        - Intent 和事件相关操作
         */
    }
}
```

#### Context 的对比

```
             Application    Activity      Service
─────────────────────────────────────────────────
生命周期    应用级         Activity生命周期 Service生命周期
UI 操作    ❌             ✅            ❌
保存引用   ✅             ❌            ❌
获取方式   MyApp.getInstance()  this  this
```

#### 核心要点

```
Context 是什么
├─ 上下文对象，应用的全局环境
├─ 所有 Activity、Service、Application 都是 Context
└─ 提供资源、服务、Intent 等功能

三种常见 Context
├─ Application Context（应用级，可保存到静态变量）
├─ Activity Context（Activity 级，生命周期短）
└─ Service Context（Service 级）

使用原则
├─ UI 操作用 Activity Context
├─ 长生命周期操作用 Application Context
└─ 避免保存 Activity Context 到静态变量（内存泄漏）
```

---

### 5. IntentFilter 是什么？有哪些使用场景？匹配机制是怎样的？

#### 详细答案

IntentFilter 用于指定组件（Activity、Service、BroadcastReceiver）能够处理什么类型的 Intent。

| 匹配元素 | 说明 | 示例 |
|---------|------|------|
| **Action** | Intent 的行为 | android.intent.action.VIEW |
| **Category** | Intent 的分类 | android.intent.category.DEFAULT |
| **Data** | Intent 的数据（URI、MIME 类型） | content://、*.pdf |
| **Priority** | 优先级（广播接收器） | 0-1000 |

#### 代码示例

**在 AndroidManifest.xml 中声明**

```xml
<!-- Activity 的 IntentFilter -->
<activity android:name=".MainActivity">
    <intent-filter>
        <!-- Action：应用首页 -->
        <action android:name="android.intent.action.MAIN" />

        <!-- Category：应用启动器中的图标 -->
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>

<!-- 可以处理打开 PDF 文件 -->
<activity android:name=".PdfViewerActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="application/pdf" />
    </intent-filter>
</activity>

<!-- 可以处理打开 http 链接 -->
<activity android:name=".BrowserActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" android:host="example.com" />
    </intent-filter>
</activity>

<!-- 广播接收器 -->
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter android:priority="10">
        <action android:name="com.example.MY_BROADCAST" />
    </intent-filter>
</receiver>

<!-- 接收自定义的 Intent -->
<activity android:name=".CustomActivity">
    <intent-filter>
        <action android:name="com.example.CUSTOM_ACTION" />
        <category android:name="android.intent.category.DEFAULT" />
        <data
            android:scheme="content"
            android:host="com.example.provider"
            android:pathPattern="/.*" />
    </intent-filter>
</activity>
```

**编程方式创建 IntentFilter**

```java
public class IntentFilterDemo {

    // 1. 动态创建 IntentFilter
    public void createIntentFilter() {
        IntentFilter filter = new IntentFilter();

        // 添加 Action
        filter.addAction(Intent.ACTION_VIEW);
        filter.addAction("com.example.CUSTOM_ACTION");

        // 添加 Category
        filter.addCategory(Intent.CATEGORY_DEFAULT);
        filter.addCategory(Intent.CATEGORY_BROWSABLE);

        // 添加 Data（MIME 类型）
        filter.addDataType("application/pdf");
        filter.addDataType("text/plain");

        // 添加 Data（URI scheme）
        filter.addDataScheme("http");
        filter.addDataScheme("https");
        filter.addDataAuthority("example.com", null);
        filter.addDataPath("/path", PatternMatcher.PATTERN_PREFIX);
    }

    // 2. 匹配 Intent
    public void matchIntent(IntentFilter filter, Intent intent) {
        // 检查 Intent 是否与 IntentFilter 匹配
        int match = filter.match(
            intent.getAction(),
            intent.getType(),
            intent.getScheme(),
            intent.getData(),
            intent.getCategories(),
            "android"
        );

        switch (match) {
            case IntentFilter.NO_MATCH_ACTION:
                Log.d("Match", "Action 不匹配");
                break;
            case IntentFilter.NO_MATCH_CATEGORY:
                Log.d("Match", "Category 不匹配");
                break;
            case IntentFilter.NO_MATCH_DATA:
                Log.d("Match", "Data 不匹配");
                break;
            case IntentFilter.NO_MATCH_TYPE:
                Log.d("Match", "Type 不匹配");
                break;
            default:
                Log.d("Match", "匹配成功: " + match);
        }
    }
}
```

**实际应用案例**

```java
// 案例 1：处理浏览器链接
public class LinkHandlerActivity extends AppCompatActivity {
    // AndroidManifest.xml
    // <intent-filter>
    //     <action android:name="android.intent.action.VIEW" />
    //     <category android:name="android.intent.category.DEFAULT" />
    //     <category android:name="android.intent.category.BROWSABLE" />
    //     <data android:scheme="https" android:host="example.com" />
    // </intent-filter>

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 获取传入的 Intent
        Intent intent = getIntent();
        if (Intent.ACTION_VIEW.equals(intent.getAction())) {
            Uri uri = intent.getData();
            Log.d("Link", "打开链接: " + uri);
        }
    }
}

// 案例 2：处理分享
public class ShareActivity extends AppCompatActivity {
    // AndroidManifest.xml
    // <intent-filter>
    //     <action android:name="android.intent.action.SEND" />
    //     <category android:name="android.intent.category.DEFAULT" />
    //     <data android:mimeType="text/plain" />
    // </intent-filter>

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Intent intent = getIntent();
        if (Intent.ACTION_SEND.equals(intent.getAction())) {
            String text = intent.getStringExtra(Intent.EXTRA_TEXT);
            Log.d("Share", "分享内容: " + text);
        }
    }
}

// 案例 3：设置为默认应用
public class BrowserActivity extends AppCompatActivity {
    // AndroidManifest.xml
    // <intent-filter>
    //     <action android:name="android.intent.action.VIEW" />
    //     <category android:name="android.intent.category.DEFAULT" />
    //     <category android:name="android.intent.category.BROWSABLE" />
    //     <data android:scheme="http" />
    //     <data android:scheme="https" />
    // </intent-filter>
}
```

**IntentFilter 匹配规则**

```
1. Action 匹配
   - Intent 中至少有一个 Action 必须在 Filter 中
   - Filter 中没有指定 Action 时，任何 Action 都匹配

2. Category 匹配
   - Intent 中的所有 Category 都必须在 Filter 中找到
   - Intent 中没有 Category 时，Filter 中的 Category 无需匹配
   - 系统会自动添加 DEFAULT category（重要）

3. Data 匹配
   - Scheme、Authority、Path、Type 都要对应
   - 必须同时指定 scheme 和其他元素才能匹配

匹配流程
Action 匹配？
   ↓ 是
Category 匹配？
   ↓ 是
Data 匹配？
   ↓ 是
✅ 整体匹配
```

#### 核心要点

```
IntentFilter 是什么
├─ 用于声明组件能处理什么类型的 Intent
├─ 在 Manifest 中静态声明或动态创建
└─ 系统通过 IntentFilter 查找能处理 Intent 的组件

四个匹配元素
├─ Action：Intent 的操作（必须至少有一个匹配）
├─ Category：Intent 的分类（必须全部匹配）
├─ Data：Intent 的数据（Scheme、Host、Path、Type）
└─ Priority：优先级（广播接收器）

常见使用场景
├─ 处理链接打开（浏览器、深度链接）
├─ 处理分享（分享文字、图片）
├─ 处理文件打开（PDF、图片）
└─ 广播接收（系统事件、自定义广播）
```

---

### 6. startService 和 bindService 的区别、生命周期和使用场景

#### 详细答案

| 特性 | startService | bindService |
|------|-------------|-----------|
| **生命周期** | 独立，stopService 才停止 | 绑定到客户端，客户端销毁则销毁 |
| **通信** | 单向，无返回 | 双向，通过 Binder |
| **多客户端** | 多个客户端共用一个 | 每个客户端独立 |
| **后台服务** | ✅ 可在后台长期运行 | ❌ 客户端销毁则销毁 |
| **使用场景** | 音乐播放、下载、上传 | Activity 与 Service 通信 |

#### 代码示例

**MyService.java**

```java
public class MyService extends Service {
    private static final String TAG = "MyService";
    private final IBinder binder = new LocalBinder();
    private int count = 0;

    // LocalBinder：用于 bindService 通信
    public class LocalBinder extends Binder {
        MyService getService() {
            return MyService.this;
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "onCreate");
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d(TAG, "onStartCommand, startId = " + startId);

        // 启动后台任务
        new Thread(() -> {
            while (count < 100) {
                Log.d(TAG, "运行中... count = " + count++);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        // 返回值决定 Service 被杀死后的行为
        return START_STICKY;  // 被杀死后会重启
        // return START_NOT_STICKY;  // 被杀死后不会重启
        // return START_REDELIVER_INTENT;  // 被杀死后重启并重新传递 Intent
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "onBind");
        return binder;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy");
    }

    // Service 提供的方法，供绑定的客户端调用
    public int getCount() {
        return count;
    }

    public void setCount(int value) {
        count = value;
    }
}
```

**Activity 中的使用**

```java
public class ServiceDemoActivity extends AppCompatActivity {
    private static final String TAG = "ServiceDemoActivity";

    // ========== startService 相关 ==========

    // 启动 Service
    private void startMyService() {
        Intent intent = new Intent(this, MyService.class);
        intent.putExtra("key", "value");
        startService(intent);
        Log.d(TAG, "startService 已调用");
    }

    // 停止 Service
    private void stopMyService() {
        Intent intent = new Intent(this, MyService.class);
        stopService(intent);
        Log.d(TAG, "stopService 已调用");
    }

    // ========== bindService 相关 ==========

    private MyService.LocalBinder binder;
    private boolean isBound = false;

    private final ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG, "onServiceConnected");
            binder = (MyService.LocalBinder) service;
            isBound = true;

            // 可以调用 Service 的方法
            MyService myService = binder.getService();
            Log.d(TAG, "Service count = " + myService.getCount());
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d(TAG, "onServiceDisconnected");
            isBound = false;
        }
    };

    private void bindMyService() {
        Intent intent = new Intent(this, MyService.class);
        bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
        Log.d(TAG, "bindService 已调用");
    }

    private void unbindMyService() {
        if (isBound) {
            unbindService(serviceConnection);
            isBound = false;
            Log.d(TAG, "unbindService 已调用");
        }
    }

    // ========== 生命周期演示 ==========

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_service_demo);

        findViewById(R.id.start_service_btn).setOnClickListener(v -> startMyService());
        findViewById(R.id.stop_service_btn).setOnClickListener(v -> stopMyService());
        findViewById(R.id.bind_service_btn).setOnClickListener(v -> bindMyService());
        findViewById(R.id.unbind_service_btn).setOnClickListener(v -> unbindMyService());
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindMyService();  // Activity 销毁时解绑
    }
}
```

**AIDL 远程通信（进程间通信）**

```java
// 1. 定义 AIDL 接口（IMyAidlInterface.aidl）
// package com.example.app;
// interface IMyAidlInterface {
//     int getCount();
//     void setCount(int value);
// }

// 2. Service 实现 AIDL
public class RemoteService extends Service {
    private int count = 0;

    private final IMyAidlInterface.Stub stub = new IMyAidlInterface.Stub() {
        @Override
        public int getCount() throws RemoteException {
            return count;
        }

        @Override
        public void setCount(int value) throws RemoteException {
            count = value;
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return stub;
    }
}

// 3. 在另一个进程中使用
public class RemoteActivity extends AppCompatActivity {
    private IMyAidlInterface remoteService;

    private final ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            remoteService = IMyAidlInterface.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            remoteService = null;
        }
    };

    private void bindRemoteService() {
        Intent intent = new Intent();
        intent.setComponent(new ComponentName(
            "com.example.remote",
            "com.example.remote.RemoteService"
        ));
        bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
    }
}
```

**生命周期对比**

```
startService 生命周期
─────────────────────
onCreate()           （第一次启动时）
↓
onStartCommand()     （每次启动）
↓
onDestroy()          （停止服务时）

bindService 生命周期
─────────────────────
onCreate()           （第一次绑定时）
↓
onBind()             （绑定时，返回 IBinder）
↓
onUnbind()           （解绑时）
↓
onDestroy()          （最后一个客户端解绑时）

同时使用两种方式的生命周期
─────────────────────────────
onCreate()           （第一次，startService 或 bindService）
↓
onStartCommand()     （startService 时）
↓
onBind()             （bindService 时）
↓
onUnbind()           （unbindService 时）
↓
onDestroy()          （stopService 且所有客户端都解绑）
```

#### 核心要点

```
startService（后台服务）
├─ 生命周期独立，stopService 才停止
├─ 无双向通信，单向触发
├─ 适合音乐播放、下载、后台任务
└─ 可长期运行

bindService（绑定服务）
├─ 生命周期绑定到客户端
├─ 通过 Binder 实现双向通信
├─ 适合 Activity 与 Service 交互
└─ 客户端销毁则服务停止

选择建议
├─ 需要长期后台运行 → startService
├─ 需要 Activity 与 Service 通信 → bindService
├─ 两者都需要 → 同时使用两种方式
```

---

（由于篇幅限制，继续在下一部分...）

### 7. Service 如何进行保活？

#### 详细答案

Service 保活是指防止 Service 被系统杀死。主要有以下几种方式：

| 方式 | 级别 | 实现复杂度 | 可靠性 |
|------|------|---------|--------|
| **onStartCommand 返回值** | 低 | ⭐ | ⭐⭐ |
| **前台 Service** | 中 | ⭐⭐ | ⭐⭐⭐ |
| **粘性 Intent** | 低 | ⭐ | ⭐ |
| **守护进程** | 高 | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| **JobScheduler/WorkManager** | 中 | ⭐⭐ | ⭐⭐⭐⭐ |

#### 代码示例

```java
// 方式 1：onStartCommand 返回值
public class KeepAliveService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // START_STICKY：被杀死后自动重启
        // START_REDELIVER_INTENT：被杀死后重启并重新传递 Intent
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}

// 方式 2：前台 Service（推荐，Android 8.0+）
public class ForegroundService extends Service {
    private static final int NOTIFICATION_ID = 1;

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 创建通知
        NotificationManager manager = getSystemService(NotificationManager.class);

        // 创建通知渠道（Android 8.0+）
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel channel = new NotificationChannel(
                "keep_alive",
                "保活服务",
                NotificationManager.IMPORTANCE_DEFAULT
            );
            manager.createNotificationChannel(channel);
        }

        // 创建通知
        Notification notification = new NotificationCompat.Builder(this, "keep_alive")
            .setContentTitle("后台服务运行中")
            .setContentText("应用正在后台运行")
            .setSmallIcon(R.drawable.ic_notification)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .build();

        // 启动前台 Service
        startForeground(NOTIFICATION_ID, notification);

        // 启动后台任务
        startBackgroundWork();

        return START_STICKY;
    }

    private void startBackgroundWork() {
        new Thread(() -> {
            while (true) {
                try {
                    Thread.sleep(1000);
                    Log.d("Service", "Service 运行中...");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d("Service", "Service 销毁");
    }
}

// 方式 3：守护进程（两个 Service 互相拉起）
public class MainService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 启动守护 Service
        Intent guardIntent = new Intent(this, GuardService.class);
        startService(guardIntent);

        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        // 主 Service 销毁时，守护 Service 会重启它
    }
}

public class GuardService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 定期检查主 Service 是否运行
        new Thread(() -> {
            while (true) {
                try {
                    Thread.sleep(5000);
                    if (!isServiceRunning(MainService.class)) {
                        // 如果主 Service 已停止，重启它
                        Intent intent2 = new Intent(GuardService.this, MainService.class);
                        startService(intent2);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        return START_STICKY;
    }

    private boolean isServiceRunning(Class<?> serviceClass) {
        ActivityManager manager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        if (manager != null) {
            for (ActivityManager.RunningServiceInfo service : manager.getRunningServices(Integer.MAX_VALUE)) {
                if (serviceClass.getName().equals(service.service.getClassName())) {
                    return true;
                }
            }
        }
        return false;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}

// 方式 4：JobScheduler（Android 5.0+）
public class JobSchedulerKeepAlive {
    public static void scheduleKeepAlive(Context context) {
        JobScheduler scheduler = context.getSystemService(JobScheduler.class);

        JobInfo jobInfo = new JobInfo.Builder(1, new ComponentName(context, KeepAliveJobService.class))
            .setMinimumLatency(1000)  // 最少延迟 1 秒
            .setRequiresDeviceIdle(false)  // 不要求设备空闲
            .setPersisted(true)  // 重启后继续执行
            .build();

        scheduler.schedule(jobInfo);
    }
}

public class KeepAliveJobService extends JobService {
    @Override
    public boolean onStartJob(JobParameters params) {
        Log.d("JobService", "后台任务执行");

        // 执行后台任务
        new Thread(() -> {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            jobFinished(params, true);  // true 表示任务未完成，稍后继续
        }).start();

        return true;
    }

    @Override
    public boolean onStopJob(JobParameters params) {
        return true;
    }
}

// 方式 5：WorkManager（最现代的方案，推荐）
public class KeepAliveWorker extends Worker {
    public KeepAliveWorker(@NonNull Context context, @NonNull WorkerParameters params) {
        super(context, params);
    }

    @NonNull
    @Override
    public Result doWork() {
        Log.d("WorkManager", "后台任务运行");

        try {
            Thread.sleep(5000);
            return Result.success();
        } catch (InterruptedException e) {
            return Result.retry();
        }
    }
}

public class WorkManagerKeepAlive {
    public static void scheduleKeepAlive(Context context) {
        // 创建后台任务
        PeriodicWorkRequest keepAliveWork = new PeriodicWorkRequest.Builder(
            KeepAliveWorker.class,
            15,  // 最少间隔 15 分钟
            TimeUnit.MINUTES
        ).build();

        // 应用到系统
        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            "keep_alive",
            ExistingPeriodicWorkPolicy.KEEP,
            keepAliveWork
        );
    }
}
```

**AndroidManifest.xml 配置**

```xml
<!-- 声明 Service -->
<service
    android:name=".KeepAliveService"
    android:exported="false" />

<!-- 前台 Service 权限 -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />

<!-- Service 保活需要的权限 -->
<uses-permission android:name="android.permission.GET_TASKS" />
<uses-permission android:name="android.permission.GET_RUNNING_APPS" />
```

#### 核心要点

```
Service 保活方式
├─ START_STICKY：简单但不可靠
├─ 前台 Service：可靠，需要通知
├─ 守护进程：复杂，容易被系统限制
├─ JobScheduler：可靠，有延迟
└─ WorkManager：最推荐，功能完整

现代方案推荐
├─ Android 8.0+ → 前台 Service + WorkManager
├─ 需要高可靠性 → WorkManager
├─ 需要定时任务 → WorkManager
└─ 简单场景 → START_STICKY
```

---

### 8. ContentProvider 如何实现数据共享？

#### 详细答案

ContentProvider 是 Android 四大组件之一，用于在应用间共享数据。

#### 代码示例

```java
// 1. 创建 ContentProvider
public class UserContentProvider extends ContentProvider {
    private static final String AUTHORITY = "com.example.app.provider";
    private static final String TABLE_NAME = "users";

    private static final int USERS = 1;
    private static final int USERS_ID = 2;

    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        uriMatcher.addURI(AUTHORITY, TABLE_NAME, USERS);
        uriMatcher.addURI(AUTHORITY, TABLE_NAME + "/#", USERS_ID);
    }

    private SQLiteDatabase database;

    @Override
    public boolean onCreate() {
        DatabaseHelper helper = new DatabaseHelper(getContext());
        database = helper.getWritableDatabase();
        return database != null;
    }

    @Nullable
    @Override
    public Cursor query(@NonNull Uri uri, @Nullable String[] projection,
                       @Nullable String selection, @Nullable String[] selectionArgs,
                       @Nullable String sortOrder) {
        switch (uriMatcher.match(uri)) {
            case USERS:
                // 查询所有用户
                return database.query(TABLE_NAME, projection, selection, selectionArgs, null, null, sortOrder);
            case USERS_ID:
                // 查询单个用户
                String id = uri.getPathSegments().get(1);
                return database.query(TABLE_NAME, projection, "_id = ?", new String[]{id}, null, null, sortOrder);
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        switch (uriMatcher.match(uri)) {
            case USERS:
                return "vnd.android.cursor.dir/vnd.example.users";
            case USERS_ID:
                return "vnd.android.cursor.item/vnd.example.user";
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        long rowId = database.insert(TABLE_NAME, null, values);
        if (rowId > 0) {
            Uri newUri = ContentUris.withAppendedId(uri, rowId);
            getContext().getContentResolver().notifyChange(newUri, null);
            return newUri;
        }
        return null;
    }

    @Override
    public int update(@NonNull Uri uri, @Nullable ContentValues values,
                     @Nullable String selection, @Nullable String[] selectionArgs) {
        int count = database.update(TABLE_NAME, values, selection, selectionArgs);
        if (count > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

    @Override
    public int delete(@NonNull Uri uri, @Nullable String selection,
                     @Nullable String[] selectionArgs) {
        int count = database.delete(TABLE_NAME, selection, selectionArgs);
        if (count > 0) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }
}

// 2. 在 AndroidManifest.xml 中声明
// <provider
//     android:name=".UserContentProvider"
//     android:authorities="com.example.app.provider"
//     android:exported="true" />

// 3. 使用 ContentProvider
public class ContentProviderUsage {
    private ContentResolver contentResolver;

    public ContentProviderUsage(Context context) {
        this.contentResolver = context.getContentResolver();
    }

    // 查询
    public void queryUsers() {
        Uri uri = Uri.parse("content://com.example.app.provider/users");
        Cursor cursor = contentResolver.query(uri, null, null, null, null);

        if (cursor != null) {
            while (cursor.moveToNext()) {
                int id = cursor.getInt(cursor.getColumnIndex("_id"));
                String name = cursor.getString(cursor.getColumnIndex("name"));
                String email = cursor.getString(cursor.getColumnIndex("email"));
                Log.d("User", id + ": " + name + ", " + email);
            }
            cursor.close();
        }
    }

    // 插入
    public void insertUser(String name, String email) {
        Uri uri = Uri.parse("content://com.example.app.provider/users");
        ContentValues values = new ContentValues();
        values.put("name", name);
        values.put("email", email);

        Uri newUri = contentResolver.insert(uri, values);
        Log.d("Insert", "新用户 URI: " + newUri);
    }

    // 更新
    public void updateUser(int id, String name) {
        Uri uri = Uri.parse("content://com.example.app.provider/users/" + id);
        ContentValues values = new ContentValues();
        values.put("name", name);

        int count = contentResolver.update(uri, values, null, null);
        Log.d("Update", "更新了 " + count + " 条记录");
    }

    // 删除
    public void deleteUser(int id) {
        Uri uri = Uri.parse("content://com.example.app.provider/users/" + id);
        int count = contentResolver.delete(uri, null, null);
        Log.d("Delete", "删除了 " + count + " 条记录");
    }

    // 观察数据变化
    public void observeDataChange() {
        Uri uri = Uri.parse("content://com.example.app.provider/users");
        contentResolver.registerContentObserver(uri, true, new ContentObserver(new Handler()) {
            @Override
            public void onChange(boolean selfChange) {
                super.onChange(selfChange);
                Log.d("Observer", "数据发生变化");
            }
        });
    }
}
```

#### 核心要点

```
ContentProvider 三要素
├─ URI：content://authority/path
├─ ContentResolver：访问 ContentProvider
└─ ContentObserver：监听数据变化

主要方法
├─ query()：查询数据
├─ insert()：插入数据
├─ update()：更新数据
├─ delete()：删除数据
└─ getType()：获取 MIME 类型

应用场景
├─ 应用间数据共享
├─ 同应用不同进程共享数据
├─ 实现数据库或文件访问接口
└─ 系统 API（联系人、日历、相册）
```

---

### 9. 切换横竖屏时 Activity 的生命周期变化

#### 详细答案

默认情况下，切换横竖屏时 Activity 会销毁并重建。

**默认行为（销毁重建）**
```
横屏 → 竖屏
────────────
onPause()
↓
onStop()
↓
onDestroy()
↓ (创建新实例)
onCreate()
↓
onStart()
↓
onResume()
```

**禁用横竖屏切换**
```xml
<activity android:name=".MainActivity"
    android:screenOrientation="portrait" />  <!-- 固定竖屏 -->
<!-- android:screenOrientation="landscape" -->  <!-- 固定横屏 -->
```

**处理配置更改（保留 Activity 实例）**
```xml
<activity android:name=".MainActivity"
    android:configChanges="orientation|screenSize|keyboardHidden" />
```

#### 代码示例

```java
public class ScreenOrientationActivity extends AppCompatActivity {
    private static final String TAG = "ScreenOrientation";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.d(TAG, "onCreate");

        // 检查是否是恢复状态
        if (savedInstanceState != null) {
            Log.d(TAG, "从保存的状态恢复");
        }
    }

    @Override
    protected void onStart() {
        super.onStart();
        Log.d(TAG, "onStart");
    }

    @Override
    protected void onResume() {
        super.onResume();
        Log.d(TAG, "onResume");
    }

    @Override
    protected void onPause() {
        super.onPause();
        Log.d(TAG, "onPause");
    }

    @Override
    protected void onStop() {
        super.onStop();
        Log.d(TAG, "onStop");
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy");
    }

    @Override
    public void onConfigurationChanged(@NonNull Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        Log.d(TAG, "onConfigurationChanged");

        // 这个方法只有在 AndroidManifest.xml 中配置了
        // android:configChanges="orientation|screenSize" 时才会调用

        if (newConfig.orientation == Configuration.ORIENTATION_PORTRAIT) {
            Log.d(TAG, "切换到竖屏");
        } else if (newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE) {
            Log.d(TAG, "切换到横屏");
        }
    }

    @Override
    protected void onSaveInstanceState(@NonNull Bundle outState) {
        super.onSaveInstanceState(outState);
        // 保存数据
        outState.putString("data", "要保存的数据");
        Log.d(TAG, "onSaveInstanceState");
    }

    @Override
    protected void onRestoreInstanceState(@NonNull Bundle savedInstanceState) {
        super.onRestoreInstanceState(savedInstanceState);
        // 恢复数据
        String data = savedInstanceState.getString("data");
        Log.d(TAG, "onRestoreInstanceState: " + data);
    }
}
```

#### 核心要点

```
横竖屏切换的三种处理方式：

1. 默认行为（销毁重建）
   - 切换时 Activity 销毁重建
   - 需要在 onSaveInstanceState 保存数据

2. 固定方向
   - 在 AndroidManifest.xml 指定 screenOrientation
   - Activity 不会重建

3. 处理配置变更（推荐）
   - 配置 android:configChanges="orientation|screenSize"
   - Activity 不销毁，调用 onConfigurationChanged
   - 自己重新加载布局
```

---

### 10. Activity 中 onNewIntent 方法的调用时机和使用场景

#### 详细答案

onNewIntent 在 Activity 的启动模式为 singleTask 或 singleTop，且收到新的 Intent 时调用。

#### 代码示例

```java
public class SingleTopActivity extends AppCompatActivity {
    private static final String TAG = "SingleTopActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.d(TAG, "onCreate");

        handleIntent(getIntent());
    }

    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        Log.d(TAG, "onNewIntent");

        // ⚠️ 重要：必须调用 setIntent() 更新当前 Intent
        setIntent(intent);
        handleIntent(intent);
    }

    private void handleIntent(Intent intent) {
        if (intent != null) {
            String data = intent.getStringExtra("key");
            Log.d(TAG, "收到数据: " + data);
        }
    }
}
```

---

### 11. Intent 传输数据的大小限制

#### 详细答案

Intent 通过 Bundle 传输数据，大小限制约为 1-2MB（与 Android 版本有关）。

**超大数据传输方案：**
- 使用文件（写入文件，传文件路径）
- 使用全局变量或单例
- 使用数据库或 ContentProvider
- 使用 Parcelable 优化

#### 代码示例

```java
// ❌ 不推荐：直接传输大数据
public void badPractice(byte[] largeData) {
    Intent intent = new Intent(this, AnotherActivity.class);
    intent.putExtra("data", largeData);  // 可能超过限制
    startActivity(intent);
}

// ✓ 推荐方案 1：使用文件
public void goodPractice1(byte[] largeData) throws IOException {
    // 写入文件
    File file = new File(getCacheDir(), "temp_data.bin");
    FileOutputStream fos = new FileOutputStream(file);
    fos.write(largeData);
    fos.close();

    // 传递文件路径
    Intent intent = new Intent(this, AnotherActivity.class);
    intent.putExtra("file_path", file.getAbsolutePath());
    startActivity(intent);
}

// ✓ 推荐方案 2：使用全局单例
public class DataHolder {
    private static DataHolder instance;
    private byte[] data;

    public static DataHolder getInstance() {
        if (instance == null) {
            instance = new DataHolder();
        }
        return instance;
    }

    public void setData(byte[] data) {
        this.data = data;
    }

    public byte[] getData() {
        return data;
    }
}

public void goodPractice2(byte[] largeData) {
    // 保存到全局对象
    DataHolder.getInstance().setData(largeData);

    // 只传递标记
    Intent intent = new Intent(this, AnotherActivity.class);
    intent.putExtra("has_data", true);
    startActivity(intent);
}
```

---

### 12. ContentProvider、ContentResolver、ContentObserver 之间的关系

#### 详细答案

| 组件 | 职责 |
|------|------|
| **ContentProvider** | 数据提供者，实现数据访问接口 |
| **ContentResolver** | 数据消费者，访问 ContentProvider |
| **ContentObserver** | 观察者，监听数据变化 |

```
应用 A                          应用 B
│                              │
├─ ContentProvider             ├─ ContentResolver
│  (数据提供)                   │  (访问数据)
│         ↑                     │  ↓
└─────────┼─────────────────────┼──────
          │                     │
     共享数据库或文件
```

---

### 13. Activity 加载的流程

#### 详细答案

Activity 启动是一个复杂的过程，涉及 ActivityManagerService、Zygote 等多个系统组件。

**简化流程：**
```
startActivity(Intent)
    ↓
AMS.startActivity()
    ↓
检查权限、解析 Intent
    ↓
创建 ActivityRecord
    ↓
启动新进程（如果需要）
    ↓
执行 Instrumentation.callActivityOnCreate()
    ↓
Activity.onCreate()
    ↓
Activity.onStart()
    ↓
Activity.onResume()
    ↓
显示在屏幕上
```

---

## 📌 Android 异步消息机制

### 1. HandlerThread 的使用场景和实现原理

#### 详细答案

HandlerThread 是一个内置了 Looper 的线程，用于处理异步任务。

#### 代码示例

```java
// 1. 创建和使用 HandlerThread
public class HandlerThreadDemo {
    private HandlerThread handlerThread;
    private Handler handler;

    public void startHandlerThread() {
        // 创建 HandlerThread
        handlerThread = new HandlerThread("BackgroundThread");
        handlerThread.start();  // ⚠️ 必须调用 start()

        // 创建 Handler，使用 HandlerThread 的 Looper
        handler = new Handler(handlerThread.getLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                super.handleMessage(msg);
                Log.d("HandlerThread", "处理消息: " + msg.what);
                // 这里运行在后台线程中
            }
        };
    }

    public void sendTask() {
        // 发送消息到后台线程
        Message msg = Message.obtain();
        msg.what = 1;
        msg.obj = "Task Data";
        handler.sendMessage(msg);

        // 或使用 post
        handler.post(() -> {
            Log.d("HandlerThread", "在后台线程执行");
        });
    }

    public void stopHandlerThread() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
            handlerThread.quitSafely();  // 安全退出
        } else {
            handlerThread.quit();
        }
    }
}

// 2. HandlerThread 实现原理
public class HandlerThreadImplementation extends Thread {
    private Looper looper;

    @Override
    public void run() {
        Looper.prepare();  // 为线程创建 Looper
        looper = Looper.myLooper();  // 获取当前线程的 Looper
        Looper.loop();  // 开启消息循环
    }

    public Looper getLooper() {
        // 等待 Looper 创建完成
        synchronized (this) {
            while (looper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {}
            }
        }
        return looper;
    }
}
```

#### 核心要点

```
HandlerThread
├─ 本质：内置 Looper 的 Thread
├─ 用途：处理后台异步任务
├─ 优点：避免重复创建线程
└─ 缺点：一次只能处理一个任务

使用步骤
├─ 创建 HandlerThread
├─ 调用 start()
├─ 基于其 Looper 创建 Handler
└─ 通过 Handler 发送消息
```

---

### 2. IntentService 的应用场景和实现原理

#### 详细答案

IntentService 是一个专门用于处理异步任务的 Service，内部使用 HandlerThread。

#### 代码示例

```java
// 1. 自定义 IntentService
public class DownloadService extends IntentService {
    private static final String TAG = "DownloadService";
    private static final String ACTION_DOWNLOAD = "com.example.action.DOWNLOAD";

    public DownloadService() {
        super("DownloadService");
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        // 这个方法运行在后台线程中
        if (intent != null) {
            String action = intent.getAction();
            if (ACTION_DOWNLOAD.equals(action)) {
                String url = intent.getStringExtra("url");
                downloadFile(url);
            }
        }
    }

    private void downloadFile(String url) {
        Log.d(TAG, "开始下载: " + url);
        try {
            Thread.sleep(3000);  // 模拟下载
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "下载完成");
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "IntentService 销毁");
    }
}

// 2. 使用 IntentService
public class MainActivity extends AppCompatActivity {
    public void startDownload() {
        Intent intent = new Intent(this, DownloadService.class);
        intent.setAction("com.example.action.DOWNLOAD");
        intent.putExtra("url", "https://example.com/file.apk");
        startService(intent);
    }
}

// 3. IntentService 实现原理
public class IntentServiceImplementation extends Service {
    private volatile Looper looper;
    private volatile ServiceHandler handler;
    private String name;

    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent) msg.obj);
            stopSelf(msg.arg1);  // 处理完成后自动停止 Service
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService");
        thread.start();
        looper = thread.getLooper();
        handler = new ServiceHandler(looper);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        Message msg = handler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        handler.sendMessage(msg);
        return START_REDELIVER_INTENT;
    }

    protected void onHandleIntent(@Nullable Intent intent) {
        // 子类实现
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

---

### 3. AsyncTask 的优点、缺点和内部实现原理

#### 详细答案

AsyncTask 用于在后台线程执行任务，并在主线程更新 UI。

| 特性 | 优点 | 缺点 |
|------|------|------|
| **使用简单** | ✓ 易于理解 | ❌ 容易内存泄漏 |
| **生命周期** | ✗ | ❌ 生命周期绑定 Task |
| **性能** | ❌ 线程池有上限 | ❌ 不适合大量并发 |
| **API** | ✓ 原生 API | ❌ 已过时（已废弃） |

#### 代码示例

```java
// 1. 使用 AsyncTask
public class MyAsyncTask extends AsyncTask<String, Integer, String> {
    private static final String TAG = "MyAsyncTask";

    // 前台线程，准备工作
    @Override
    protected void onPreExecute() {
        super.onPreExecute();
        Log.d(TAG, "onPreExecute - 主线程");
        // 显示进度条
    }

    // 后台线程，耗时操作
    @Override
    protected String doInBackground(String... params) {
        Log.d(TAG, "doInBackground - 后台线程");
        String url = params[0];

        for (int i = 0; i <= 100; i++) {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 发送进度更新到主线程
            publishProgress(i);
        }

        return "下载完成";
    }

    // 主线程，更新 UI
    @Override
    protected void onProgressUpdate(Integer... values) {
        super.onProgressUpdate(values);
        Log.d(TAG, "onProgressUpdate - 主线程, 进度: " + values[0]);
        // 更新进度条
    }

    // 主线程，处理结果
    @Override
    protected void onPostExecute(String result) {
        super.onPostExecute(result);
        Log.d(TAG, "onPostExecute - 主线程, 结果: " + result);
        // 显示下载完成
    }

    // 取消任务时调用
    @Override
    protected void onCancelled() {
        super.onCancelled();
        Log.d(TAG, "onCancelled");
    }
}

// 2. 启动 AsyncTask
public class AsyncTaskDemo {
    public void startTask() {
        MyAsyncTask task = new MyAsyncTask();
        task.execute("http://example.com/file");

        // 或者取消任务
        // task.cancel(true);
    }
}

// 3. AsyncTask 的线程限制（重要）
public class AsyncTaskThreadPool {
    /*
    AsyncTask 使用线程池，有数量限制：
    - 核心线程数：CPU 核心数 + 1
    - 最大线程数：CPU 核心数 * 2 + 1
    - 队列大小：128

    如果超过限制，会抛出 RejectedExecutionException
    */
}
```

**⚠️ 为什么 AsyncTask 已过时：**
- 容易导致内存泄漏（持有 Activity 引用）
- 线程池有限制
- 不能正确处理 Activity 销毁后的更新
- **推荐使用 LiveData + ViewModel 或 RxJava**

---

### 4. Activity.runOnUiThread 的理解

#### 详细答案

runOnUiThread 用于在任意线程中更新 UI，它会自动切换到主线程。

#### 代码示例

```java
public class RunOnUiThreadDemo {
    // ❌ 错误：在后台线程更新 UI（会崩溃）
    public void badPractice() {
        new Thread(() -> {
            // android.view.ViewRootImpl$CalledFromWrongThreadException
            // textView.setText("更新失败");
        }).start();
    }

    // ✓ 正确：使用 runOnUiThread
    public void goodPractice(Activity activity) {
        new Thread(() -> {
            // 在后台线程执行耗时操作
            String result = downloadData();

            // 切换到主线程更新 UI
            activity.runOnUiThread(() -> {
                // 这里运行在主线程
                // textView.setText(result);
            });
        }).start();
    }

    private String downloadData() {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "下载完成";
    }
}

// runOnUiThread 的实现原理
public class RunOnUiThreadImplementation {
    /*
    public final void runOnUiThread(Runnable action) {
        if (Thread.currentThread() != mUiThread) {
            // 当前不是主线程，发送 Message 到主线程
            mHandler.post(action);
        } else {
            // 当前已是主线程，直接执行
            action.run();
        }
    }
    */
}
```

---

### 5. Android 的子线程能否更新 UI？

#### 详细答案

**简单回答：不能。**
- Android 要求 UI 更新只能在主线程进行
- 其他线程更新 UI 会抛出 `CalledFromWrongThreadException` 异常

**但有例外：**
- 在 `Window` 被 `add` 之前，任何线程都可以更新 UI
- 例如：在 `onCreate` 中的 `setContentView` 之前创建的 View

#### 代码示例

```java
public class ThreadUIDemo {
    // ❌ 会崩溃
    public void wrongWay() {
        new Thread(() -> {
            TextView tv = findViewById(R.id.tv);
            // android.view.ViewRootImpl$CalledFromWrongThreadException
            tv.setText("From Background Thread");
        }).start();
    }

    // ✓ 正确方式 1：Handler
    public void correctWay1() {
        Handler handler = new Handler(Looper.getMainLooper());
        new Thread(() -> {
            handler.post(() -> {
                // 在主线程更新 UI
            });
        }).start();
    }

    // ✓ 正确方式 2：View.post()
    public void correctWay2(View view) {
        new Thread(() -> {
            view.post(() -> {
                // 在主线程更新 UI
            });
        }).start();
    }

    // ✓ 正确方式 3：runOnUiThread
    public void correctWay3(Activity activity) {
        new Thread(() -> {
            activity.runOnUiThread(() -> {
                // 在主线程更新 UI
            });
        }).start();
    }

    // ⚠️ 例外：在 Window 添加前
    public void exception() {
        new Thread(() -> {
            // 这是安全的，因为还没有 add 到 Window
            // 但不推荐这样做
        }).start();
    }
}
```

---

### 6. Android 中消息机制和原理

#### 详细答案

Android 消息机制是由 **Message、MessageQueue、Looper、Handler** 四部分组成。

**工作流程：**
```
Handler.sendMessage(msg)
    ↓
MessageQueue.enqueueMessage(msg)  // 入队
    ↓
Looper.loop()  // 不断轮询
    ↓
MessageQueue.next()  // 出队
    ↓
Handler.dispatchMessage(msg)  // 分发
    ↓
Handler.handleMessage(msg)  // 处理
```

#### 代码示例

```java
public class MessageMechanismDemo {
    private static final String TAG = "MessageMechanism";

    // 1. Handler 的创建和使用
    public static class MyHandler extends Handler {
        @Override
        public void handleMessage(@NonNull Message msg) {
            super.handleMessage(msg);
            Log.d(TAG, "处理消息: " + msg.what);
            switch (msg.what) {
                case 1:
                    Log.d(TAG, "收到消息 1: " + msg.obj);
                    break;
                case 2:
                    Log.d(TAG, "收到消息 2: " + msg.arg1 + ", " + msg.arg2);
                    break;
            }
        }
    }

    // 2. 消息机制详解
    public static class DetailedMechanism {

        public void sendMessage() {
            Handler handler = new Handler(Looper.getMainLooper());

            // 方式 1：发送 Message
            Message msg = Message.obtain();
            msg.what = 1;
            msg.obj = "数据";
            handler.sendMessage(msg);

            // 方式 2：延迟发送
            handler.sendMessageDelayed(msg, 2000);  // 延迟 2 秒

            // 方式 3：发送 Runnable
            handler.post(() -> {
                Log.d(TAG, "在主线程执行");
            });

            // 方式 4：延迟发送 Runnable
            handler.postDelayed(() -> {
                Log.d(TAG, "延迟执行");
            }, 2000);
        }

        public void looperMechanism() {
            // Looper 原理
            /*
            public static void loop() {
                final Looper me = myLooper();
                final MessageQueue queue = me.mQueue;

                while (true) {
                    Message msg = queue.next();  // 阻塞操作
                    if (msg == null) {
                        return;
                    }

                    msg.target.dispatchMessage(msg);  // msg.target 是 Handler
                }
            }
            */
        }

        public void messageQueueMechanism() {
            // MessageQueue 的队列数据结构
            /*
            Message 形成链表
            mMessages → Message1 → Message2 → Message3 → null

            enqueueMessage 根据延迟时间插入合适位置（小顶堆）
            */
        }
    }

    // 3. Handler、Looper、Thread 的关系
    public static class ThreadAndHandler {
        public void relationshipDemo() {
            // 主线程已有 Looper
            Handler mainHandler = new Handler(Looper.getMainLooper());

            // 在后台线程中
            new Thread("BackgroundThread") {
                @Override
                public void run() {
                    // 创建当前线程的 Looper
                    Looper.prepare();

                    Handler backgroundHandler = new Handler(Looper.myLooper()) {
                        @Override
                        public void handleMessage(@NonNull Message msg) {
                            Log.d(TAG, "后台线程处理: " + msg.what);
                        }
                    };

                    // 开启消息循环
                    Looper.loop();
                }
            }.start();
        }
    }

    // 4. Message 对象
    public static class MessageDetail {
        public void messageFields() {
            Message msg = Message.obtain();
            msg.what = 1;  // 消息类型
            msg.arg1 = 10;  // 整数参数 1
            msg.arg2 = 20;  // 整数参数 2
            msg.obj = "数据";  // 任意对象
            msg.replyTo = null;  // 回复 Handler
        }
    }
}
```

---

### 7. 为什么在子线程中创建 Handler 会抛异常？

#### 详细答案

因为子线程默认没有 Looper。Handler 需要 Looper 来获取消息。

#### 代码示例

```java
public class HandlerInThreadDemo {

    // ❌ 错误：子线程中直接创建 Handler
    public void wrongWay() {
        new Thread(() -> {
            // java.lang.RuntimeException: Can't create handler inside thread
            // that has not called Looper.prepare()
            Handler handler = new Handler();  // 崩溃
        }).start();
    }

    // ✓ 正确：先创建 Looper
    public void correctWay() {
        new Thread(() -> {
            Looper.prepare();  // 为线程创建 Looper
            Handler handler = new Handler();  // 现在可以创建 Handler
            Looper.loop();  // 启动消息循环
        }).start();
    }

    // ✓ 更简洁：使用 HandlerThread
    public void betterWay() {
        HandlerThread thread = new HandlerThread("MyThread");
        thread.start();
        Handler handler = new Handler(thread.getLooper());
    }
}
```

---

### 8. Handler 的 post 和 sendMessage 方法的区别和应用场景

#### 详细答案

| 方法 | 参数 | 场景 | 是否可取消 |
|------|------|------|---------|
| **post(Runnable)** | Runnable | 简单任务 | ✓ |
| **sendMessage(Message)** | Message | 复杂消息 | ✓ |

两者本质相同，post 最终也会被转换为 Message。

#### 代码示例

```java
public class PostVsSendMessage {
    private Handler handler = new Handler(Looper.getMainLooper());

    // post 方式
    public void postWay() {
        handler.post(() -> {
            Log.d("Handler", "post 执行");
        });

        // 延迟
        handler.postDelayed(() -> {
            Log.d("Handler", "延迟执行");
        }, 2000);

        // 移除
        handler.removeCallbacks(() -> {});
    }

    // sendMessage 方式
    public void sendMessageWay() {
        Message msg = Message.obtain();
        msg.what = 1;
        msg.obj = "数据";
        handler.sendMessage(msg);

        // 延迟
        handler.sendMessageDelayed(msg, 2000);

        // 移除
        handler.removeMessages(1);
    }

    // post 的内部实现
    private class PostImplementation {
        /*
        public final boolean post(Runnable r) {
            return sendMessageDelayed(getPostMessage(r), 0);
        }

        private Message getPostMessage(Runnable r) {
            Message m = Message.obtain();
            m.callback = r;  // 保存 Runnable
            return m;
        }

        public void dispatchMessage(Message msg) {
            if (msg.callback != null) {
                msg.callback.run();  // 执行 Runnable
            } else {
                handleMessage(msg);  // 或调用 handleMessage
            }
        }
        */
    }
}
```

---

### 9. Handler 中有 Loop 死循环，为什么没有阻塞主线程？

#### 详细答案

**答案：** Looper.loop() 不是真正的忙轮询，而是基于 **Linux epoll 的事件驱动机制**。

**工作原理：**
1. MessageQueue 为空时，线程进入阻塞状态（等待事件）
2. 有消息到达时，从阻塞状态唤醒
3. 处理消息后，继续阻塞等待
4. 这样看起来像死循环，但实际不消耗 CPU

#### 代码示例

```java
public class LooperDeadLoopDemo {

    // Looper.loop() 的简化实现
    public static class SimpleLooperImplementation {
        /*
        public static void loop() {
            Looper me = myLooper();
            MessageQueue queue = me.mQueue;

            while (true) {
                Message msg = queue.next();  // 可能阻塞在这里

                if (msg == null) {
                    return;
                }

                msg.target.dispatchMessage(msg);  // 分发消息
            }
        }

        MessageQueue.next() {
            while (true) {
                synchronized (this) {
                    // 如果没有消息，等待（阻塞）
                    if (mMessages == null) {
                        wait();  // 线程进入等待状态
                    }
                    // 有消息则返回
                    if (mMessages != null) {
                        return mMessages;
                    }
                }
            }
        }
        */
    }

    // 主线程之所以不卡死的原因
    public static class WhyNotBlocking {
        /*
        1. 主线程一开始就启动了 Looper.loop()
        2. Looper 不断轮询 MessageQueue
        3. 如果没有消息，线程阻塞（等待）
        4. 用户操作、系统事件产生消息
        5. MessageQueue 收到消息，唤醒线程
        6. 线程处理消息
        7. 继续轮询...

        底层机制：
        - 使用 epoll / select / poll（Linux 事件驱动）
        - Java 层通过 native 方法与 native 层通信
        - native 层使用 epoll 监听 FD
        - 有事件时唤醒，无事件时阻塞
        */
    }

    public void demonstrateNonBlocking() {
        // 不会卡死的演示
        Handler handler = new Handler(Looper.getMainLooper());

        // 发送消息到未来
        handler.sendMessageDelayed(Message.obtain(), 5000);

        // 在这 5 秒内，主线程不被占用
        // 可以处理用户输入、系统事件等
        // 5 秒后消息被处理

        // 这就是为什么看起来是死循环，但不阻塞主线程
    }
}
```

**关键理解：**
```
❌ 错误理解：
    while(true) { ... }  // 这样会阻塞 CPU

✓ 正确理解：
    while(true) {
        msg = queue.next();  // 阻塞，等待消息
        if (msg != null) {
            handle(msg);  // 处理消息
        }
    }
    // queue.next() 在没有消息时会阻塞（sleep），不占用 CPU
    // 等有消息时被唤醒
```

---

## 📊 学习进度总结

### 已完成（Part 1：22 题）
✅ **Android 四大组件（13 题）**
- Activity 与 Fragment 通信
- LaunchMode 启动模式
- BroadcastReceiver 与 LocalBroadcastManager
- Context 详解
- IntentFilter 匹配机制
- startService vs bindService
- Service 保活
- ContentProvider 数据共享
- Activity 横竖屏生命周期
- onNewIntent 方法
- Intent 数据大小限制
- ContentProvider / ContentResolver / ContentObserver
- Activity 加载流程

✅ **Android 异步消息机制（9 题）**
- HandlerThread
- IntentService
- AsyncTask
- runOnUiThread
- 子线程更新 UI
- 消息机制原理
- 子线程创建 Handler
- Handler post vs sendMessage
- Looper 死循环

### 后续计划（Part 2 + 3）
⏳ **Android UI 绘制（28 题）**
⏳ **Android 性能优化（17 题）**
⏳ **Android IPC / SDK / 框架 / 综合（50 题）**

---

🎉 **Part 1 已完成！准备好学习 Part 2 吗？**





