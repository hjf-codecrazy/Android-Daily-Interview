# Java 快速复习卡

> 每题 3-5 个关键词 + 核心要点，适合快速背诵和考前冲刺

---

## 🎯 Java 基础部分

### 1️⃣ 抽象类与接口的区别

**关键词**：IS-A | 单继承 | CAN-DO | 多实现 | 构造方法

**核心要点**：
- 抽象类：表示"是一个"关系，单继承，有构造方法，可以有具体方法
- 接口：表示"能够做"关系，多实现，没有构造方法，Java8+ 可有默认方法
- **选择**：相同特征用抽象类，功能拓展用接口

---

### 2️⃣ final、static、synchronized 的修饰范围和作用

**关键词**：不可变 | 全局共享 | 互斥锁 | 编译时 | 运行时

**核心要点**：
- **final**：修饰类（不能继承）、方法（不能重写）、变量（只能赋值一次）
- **static**：修饰变量（全局共享）、方法（可通过类名调用）、代码块（加载时执行）
- **synchronized**：修饰方法/代码块，实现互斥锁，保证线程安全

**快速记忆**：
```
final  → 不改
static → 全局共享
synchronized → 互斥锁
```

---

### 3️⃣ String、StringBuffer、StringBuilder 的区别

**关键词**：不可变 | 线程安全 | 性能 | 拼接字符串 | 多线程

**核心要点**：
- String：**不可变**，频繁创建新对象，低性能
- StringBuffer：**可变，线程安全**，有 synchronized，性能中等
- StringBuilder：**可变，线程不安全**，性能最好

**性能排序**：StringBuilder > StringBuffer > String（频繁拼接时）

**使用场景**：
- 不需要修改 → String
- 多线程拼接 → StringBuffer
- 单线程拼接 → StringBuilder

---

### 4️⃣ equals()、== 和 hashCode() 的区别

**关键词**：引用比较 | 内容比较 | 哈希值 | HashMap | HashSet

**核心要点**：
- **==**：比较引用是否相同（地址）
- **equals()**：比较对象内容是否相同（可重写）
- **hashCode()**：返回对象的哈希值（HashMap/HashSet key）

**关键规则**：
```
equals() 相等 → hashCode() 必须相等
hashCode() 相等 → equals() 不一定相等
```

**必须同时重写**：保证 HashMap/HashSet 正确使用

---

### 5️⃣ 深拷贝与浅拷贝

**关键词**：引用复制 | 对象复制 | 相互影响 | 内存占用 | 独立性

**核心要点**：
- **浅拷贝**：复制引用，修改会相互影响
- **深拷贝**：复制对象内容，完全独立

**实现方式**：
- 浅拷贝：`Object.clone()`
- 深拷贝：手动复制 | 序列化反序列化

**何时选择**：
- 对象内有引用类型 → 必须深拷贝
- 只有基本类型 → 浅拷贝可以

---

### 6️⃣ Error 和 Exception 的区别

**关键词**：不可恢复 | 可恢复 | 运行时 | StackOverflow | IOException

**核心要点**：
- **Error**：系统级错误，**不可恢复**，通常不捕获
- **Exception**：程序异常，**可恢复**，应该捕获处理

**分类**：
```
Throwable
├── Error（StackOverflowError、OutOfMemoryError）
└── Exception
    ├── Checked Exception（IOException、SQLException）必须处理
    └── Runtime Exception（NullPointerException、ClassCastException）可选处理
```

---

### 7️⃣ 反射机制及应用场景

**关键词**：运行时获取 | 动态调用 | Class 对象 | 框架 | 序列化

**核心要点**：
- 反射：在运行时动态获取类信息和调用方法
- 三种获取 Class 对象：`Class.forName()` | `ClassName.class` | `instance.getClass()`

**主要功能**：
```
获取 → 字段、方法、构造器
调用 → 创建对象、调用方法、修改字段
```

**应用场景**：
- 框架（Spring、Hibernate）
- 序列化/反序列化（JSON 解析）
- 动态代理、ORM 映射、注解处理

---

### 8️⃣ 重写 equals() 和 hashCode()

**关键词**：五大原则 | 自反性 | 对称性 | 传递性 | 一致性

**核心要点**：
- **equals() 五大原则**：自反性、对称性、传递性、一致性、与 null 比较
- **hashCode 契约**：equals() 相等 → hashCode() 必须相等

**正确实现**：
```java
@Override
public boolean equals(Object obj) {
    if (this == obj) return true;  // 自反性
    if (obj == null || !(obj instanceof Class)) return false;
    Class other = (Class) obj;
    return this.field1.equals(other.field1) && this.field2 == other.field2;
}

@Override
public int hashCode() {
    return Objects.hash(field1, field2);  // 同时重写
}
```

---

### 9️⃣ Java IO 流分类

**关键词**：输入输出 | 字节字符 | 节点处理 | 缓冲 | 序列化

**核心分类**：
```
按流向：InputStream / OutputStream（字节）
       Reader / Writer（字符）

按功能：FileStream / StringStream / ObjectStream
       BufferedStream / DataStream

使用建议：
- 文本数据 → 字符流（Reader/Writer）
- 二进制数据 → 字节流（InputStream/OutputStream）
- 频繁访问 → 缓冲流（BufferedXXX）
- 对象序列化 → ObjectInputStream/ObjectOutputStream
```

**性能提示**：缓冲流 > 直接流（快 10-100 倍）

---

### 🔟 Java 泛型中的类型擦除

**关键词**：编译时检查 | 运行时擦除 | 兼容性 | Object | 局限性

**核心要点**：
- 泛型在**编译时**进行类型检查，**运行时**被擦除为 Object
- 目的：保持与旧版本 Java 的兼容性

**局限性**：
```
❌ 不能创建泛型数组
❌ 不能通过 instanceof 检查泛型
❌ 不能通过反射获取泛型参数
❌ 泛型类所有实例运行时是同一个类
```

**解决方案**：TypeToken | 继承泛型类保存信息

---

### 1️⃣1️⃣ String 为什么不可变

**关键词**：安全性 | 性能 | 线程安全 | 哈希值缓存 | 字符串池

**核心要点**：
- **安全**：防止作为 HashMap key 被修改
- **性能**：可以缓存和重用，字符串池中复用
- **线程安全**：不可变对象天生线程安全
- **哈希值缓存**：加快 HashMap 查询速度

**实现原理**：
- `private final char[] value`
- 没有修改 value 的方法
- 返回新对象而非修改原对象

---

### 1️⃣2️⃣ Java 注解的理解

**关键词**：元数据 | 运行时 | 反射处理 | 框架 | @Override

**核心要点**：
- 注解是元数据，用于**提供编译器和运行时信息**
- 三个层次：编译时 | 加载时 | 运行时
- 四个元注解：@Target | @Retention | @Inherited | @Documented

**应用场景**：
```
@Override → 编译时检查
@Deprecated → 版本标记
@FunctionalInterface → 函数式接口标记
自定义注解 + 反射 → ORM、AOP、依赖注入
```

---

### 1️⃣3️⃣ 成员、局部、静态变量的生命周期

**关键词**：创建时机 | 回收时机 | 初始值 | 作用域 | GC

**核心对比**：
```
                创建        回收        初始值    作用域
成员变量        对象创建    对象GC      有        整个对象
局部变量        执行到该行  方法结束    无        该方法
静态变量        类加载      类卸载      有        整个程序
```

**执行顺序**：
```
1. 静态初始化块（类加载时）
2. 非静态初始化块（对象创建时）
3. 构造方法（对象创建时）
```

---

### 1️⃣4️⃣ String.length() 的原理

**关键词**：char 数组 | 字符数 | 字节数 | 编码 | O(1)

**核心要点**：
- `String.length()` 返回**字符数**，不是字节数
- 实现：直接返回内部 `char[] value` 的长度
- 时间复杂度：**O(1)**（直接返回，不需要遍历）

**常见混淆**：
```
String str = "你";
str.length();           // 1（字符数）
str.getBytes("UTF-8").length;  // 3（字节数）
```

**Emoji 处理**：
- 某些 Emoji 占用 2 个 char
- 用 `codePointCount()` 获取实际字符数

---

---

## 📚 Java 集合部分

### 1️⃣ List、Set、Map 的区别

**关键词**：有序 | 重复 | Key-Value | 实现类 | 使用场景

**核心对比**：
```
        顺序    重复    实现              使用场景
List    有序    允许    ArrayList        需要顺序和随机访问
        有序    允许    LinkedList       需要频繁插入删除
Set     无序    不允许  HashSet          需要不重复集合
        有序    不允许  LinkedHashSet    需要有序不重复
        排序    不允许  TreeSet          需要排序集合
Map     无序    -      HashMap           快速 Key-Value 查询
        有序    -      LinkedHashMap     需要插入顺序
```

**快速选择**：
- 有顺序 + 随机访问 → ArrayList
- 频繁插入删除 → LinkedList
- 不重复 + 快速查询 → HashSet
- 不重复 + 保证顺序 → LinkedHashSet
- Key-Value 映射 → HashMap

---

### 2️⃣ ArrayList vs LinkedList

**关键词**：数组 | 链表 | 随机访问 | 插入删除 | 性能

**核心对比**：
```
        ArrayList   LinkedList
访问    O(1)快      O(n)慢
末尾插入 O(1)       O(1)
中间插入 O(n)       O(1)快
删除    O(n)        O(n)
扩容    1.5倍       -
内存    较少        较多
```

**使用选择**：
- **ArrayList**：频繁随机访问
- **LinkedList**：频繁中间插入删除

**扩容机制**：ArrayList 满时扩容为 1.5 倍

---

### 3️⃣ HashMap vs HashTable

**关键词**：线程安全 | 性能 | null | 遗留 | 替代

**核心区别**：
```
特性              HashMap      HashTable
线程安全          否           是（synchronized）
null key/value    允许         不允许
性能              快           慢（同步开销）
继承              AbstractMap  Dictionary（遗留）
替代              -            ConcurrentHashMap
```

**现代方案**：
- 单线程 → HashMap
- 多线程 → ConcurrentHashMap（优于 HashTable）

---

### 4️⃣ ArrayList 的扩容机制

**关键词**：初始化 | 扩容系数 | System.arraycopy | 动态数组

**核心机制**：
```
1. 初始容量：10（默认）
2. 扩容时机：size == capacity 时
3. 扩容系数：1.5 倍（oldCapacity + oldCapacity >> 1）
4. 复制方式：System.arraycopy()（native 方法，高效）
```

**性能优化**：
- 如果知道大小，用 `new ArrayList<>(size)` 避免扩容
- 扩容是一个耗时操作（涉及数组复制）

---

### 5️⃣ HashMap 的实现原理

**关键词**：哈希表 | 链表 | 红黑树 | hash 冲突 | 扩容

**核心结构**：
```
数组 + 链表（哈希冲突）
↓
Java8+ 改进：数组 + 链表 + 红黑树（链表长度 > 8 转为红黑树）
```

**关键概念**：
```
1. hash() 函数：将 key 映射到数组索引
2. hash 冲突：多个 key 映射到同一索引（用链表解决）
3. 扩容：2 倍扩容，重新 hash 所有元素
4. 负载因子：0.75（默认），size > capacity * 0.75 时扩容
5. 红黑树：链表长度 > 8 且数组长度 > 64 时转为红黑树
```

**时间复杂度**：
- 最好情况：O(1)
- 最坏情况：O(n)（但红黑树优化后为 O(log n)）

---

### 6️⃣ LinkedHashMap 的原理和使用场景

**关键词**：插入顺序 | 访问顺序 | 双向链表 | LRU | 缓存

**核心特点**：
- 继承 HashMap，额外维护**双向链表**记录顺序
- 两种顺序：**插入顺序**（默认）| **访问顺序**

**使用场景**：
```
1. 需要保持插入顺序的 Map
2. LRU 缓存实现（accessOrder=true）
3. 遍历顺序需要确定的场景
```

**LRU 缓存实现**：
```java
LinkedHashMap<String, String> lruCache =
    new LinkedHashMap<String, String>(16, 0.75f, true) {  // accessOrder=true
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > 100;  // 保持最多 100 个元素
    }
};
```

---

### 7️⃣ ConcurrentHashMap 的理解

**关键词**：线程安全 | 分段锁 | 低并发 | CAS | Node

**核心设计**：
```
Java7 → 分段锁（Segment）
Java8+ → Node + CAS + synchronized（性能更优）
```

**优势**：
- 比 HashTable 高效（分段而非全表同步）
- 多线程下性能优异
- 不允许 null key/value

**使用场景**：
- 多线程访问 Map
- 不需要整体加锁

---

---

## 📚 Java 多线程部分

### 1️⃣ Java 创建多线程的方式

**关键词**：Thread 类 | Runnable 接口 | Callable | 线程池 | 优先级

**三种主要方式**：
```
1. 继承 Thread 类
   优点：简单
   缺点：不能继承其他类

2. 实现 Runnable 接口（推荐）
   优点：可以继承其他类，符合单一职责
   缺点：无返回值

3. 实现 Callable 接口 + FutureTask（Java5+）
   优点：有返回值，支持异常
   缺点：代码复杂

4. 使用线程池（最推荐）
   优点：复用线程，高效
   缺点：学习成本
```

---

### 2️⃣ 线程的几种状态

**关键词**：NEW | RUNNABLE | BLOCKED | WAITING | TERMINATED

**六种状态**：
```
NEW        → 创建但未启动
RUNNABLE   → 就绪或运行中
BLOCKED    → 等待获得锁
WAITING    → wait() 状态
TIMED_WAITING → sleep() 或 wait(timeout) 状态
TERMINATED → 线程结束
```

**状态转换**：
```
NEW → start() → RUNNABLE → BLOCKED ↔ WAITING ↔ TIMED_WAITING → TERMINATED
```

---

### 3️⃣ 如何实现多线程中的同步

**关键词**：synchronized | Lock | 互斥 | 临界区 | volatile

**同步方式**：
```
1. synchronized 关键字（隐式锁）
   - 修饰方法：锁定 this
   - 修饰代码块：指定锁对象

2. Lock 接口（显式锁，Java5+）
   - ReentrantLock（可重入锁）
   - 更灵活，支持中断、超时、条件变量

3. volatile 关键字
   - 保证可见性，不保证原子性
   - 用于标志位，不用于计数

4. Atomic 原子类
   - AtomicInteger、AtomicReference
   - CAS 操作，无锁并发
```

**性能对比**：无同步 > AtomicInteger > Lock > synchronized

---

### 4️⃣ 线程死锁及避免

**关键词**：相互等待 | 环形等待 | 四个条件 | 破坏 | 避免

**死锁的四个必要条件**：
```
1. 互斥：资源只能一个线程使用
2. 持有并等待：持有资源同时等待其他资源
3. 不可抢占：资源不能被强制抢占
4. 环形等待：线程形成环形等待
```

**破坏条件**（避免死锁）：
```
✓ 破坏互斥：资源共享（不可行）
✓ 破坏持有并等待：一次获取所有资源
✓ 破坏不可抢占：允许强制释放
✓ 破坏环形等待：资源有序分配（最常用）
```

**实践建议**：
- 避免嵌套加锁
- 避免在持有锁时调用其他方法
- 按固定顺序获取多个锁
- 使用超时机制

---

### 5️⃣ 线程阻塞的原因

**关键词**：wait | sleep | IO | 锁 | 资源

**常见阻塞场景**：
```
1. wait()：等待唤醒（释放锁）
2. sleep()：睡眠指定时间（不释放锁）
3. join()：等待其他线程结束
4. 获取锁：等待获得 synchronized 或 Lock
5. IO 操作：等待 IO 完成
6. 资源不可用：等待资源就绪
```

**区别**：
- **wait** 释放锁，**sleep/yield** 不释放
- **wait** 需要 synchronized，**sleep** 不需要

---

### 6️⃣ Thread 的 run() 和 start() 的区别

**关键词**：创建线程 | 同线程执行 | 异线程执行 | start() 一次

**核心区别**：
```
start()          执行 run()
│                │
创建新线程        当前线程执行
异步执行          同步执行
可调用多次        可以调用多次，但同一个 start() 只能调一次
正确用法          会报 IllegalThreadStateException
```

**常见误区**：
```
❌ 直接调用 run()：不会创建新线程，只是普通方法调用
✓ 调用 start()：创建新线程，异步执行 run()
```

---

### 7️⃣ synchronized 和 volatile 的区别

**关键词**：互斥 | 可见性 | 原子性 | 性能 | 使用场景

**核心区别**：
```
              synchronized     volatile
作用          互斥锁           可见性
可见性        ✓                ✓
原子性        ✓                ❌（只保证单个操作）
性能          慢               快
粒度          粗               细
场景          共享变量修改      标志位、状态
```

**使用建议**：
- 修改共享变量 → synchronized
- 标志位/状态标记 → volatile
- 计数 → Atomic 类

---

### 8️⃣ 如何保证线程安全

**关键词**：同步 | 不可变 | ThreadLocal | 线程池 | 原子类

**三个层面**：
```
1. 同步机制
   - synchronized
   - Lock / ReentrantLock
   - Atomic 原子类

2. 不可变对象
   - final 修饰字段
   - 返回副本而非引用

3. 线程隔离
   - ThreadLocal
   - 参数传递（避免共享）
   - 线程池隔离

4. 并发集合
   - ConcurrentHashMap
   - CopyOnWriteArrayList
   - ConcurrentLinkedQueue
```

---

### 9️⃣ ThreadLocal 的用法和原理

**关键词**：线程隔离 | 每个线程独立 | 内存泄漏 | 弱引用 | 上下文

**核心概念**：
```
ThreadLocal<T> → 每个线程拥有一份独立的 T 副本
不需要同步 → 避免并发问题
原理 → ThreadLocalMap（Thread 内部维护）
```

**常见应用**：
```
1. 数据库连接：Connection conn = threadLocal.get()
2. 用户上下文：User user = UserContext.get()
3. 事务管理：txManager.getTxStatus()
4. 请求追踪：RequestId 在整个请求链路中传递
```

**注意事项**：
```
⚠️ 使用完毕必须 remove()，否则内存泄漏
⚠️ 不是同步的替代品
✓ 适合用于多个方法间传递数据
```

---

### 🔟 notify 和 notifyAll 的区别

**关键词**：唤醒 | 单个 | 所有 | 竞争 | 公平性

**核心区别**：
```
notify()     → 随机唤醒一个等待线程
notifyAll()  → 唤醒所有等待线程

竞争           单个线程竞争    所有线程竞争
公平性         可能饥饿        较公平
性能           快              稍慢
使用场景       单消费者        多消费者（推荐）
```

**使用建议**：
```
❌ 避免用 notify()（可能导致线程饥饿）
✓ 优先使用 notifyAll()（更安全）
```

**示例**：
```java
// 消费者等待
synchronized(this) {
    while (isEmpty()) {
        wait();  // 等待数据
    }
    consume();
}

// 生产者通知
synchronized(this) {
    produce();
    notifyAll();  // 唤醒所有消费者
}
```

---

### 1️⃣1️⃣ 什么是线程池？如何创建

**关键词**：复用线程 | 任务队列 | 核心线程 | 拒绝策略 | ExecutorService

**核心参数**：
```
corePoolSize    → 核心线程数（始终存在）
maximumPoolSize → 最大线程数
keepAliveTime   → 空闲线程保活时间
workQueue       → 任务队列
threadFactory   → 线程创建工厂
rejectedExecutionHandler → 拒绝策略
```

**创建方式**：
```
1. ThreadPoolExecutor（推荐，参数完整）
2. Executors 工具类
   - newFixedThreadPool()：固定大小
   - newCachedThreadPool()：可缓存
   - newSingleThreadExecutor()：单线程
   - newScheduledThreadPool()：定时任务
```

**工作流程**：
```
提交任务 → 核心线程池 → 任务队列 → 创建新线程 → 拒绝
    1-5线程     队列满    5-20线程    都满
```

---

### 1️⃣2️⃣ Java 中几种常见锁

**关键词**：ReentrantLock | ReadWriteLock | StampedLock | 性能 | 使用场景

**常见锁**：
```
1. synchronized（内置锁）
   - 自动释放
   - 不可中断
   - 阻塞等待

2. ReentrantLock（可重入锁）
   - 手动释放
   - 可中断
   - 公平性选择
   - 条件变量

3. ReadWriteLock（读写锁）
   - 读操作可并发
   - 写操作互斥
   - 适合读多写少

4. StampedLock（Java8）
   - 性能更优（乐观读）
   - 使用复杂
   - 优于 ReadWriteLock

5. Atomic 原子类
   - CAS 操作
   - 无锁并发
   - 适合简单操作
```

**性能排序**：Atomic > StampedLock > ReadWriteLock > ReentrantLock > synchronized

---

### 1️⃣3️⃣ sleep() 和 wait() 的区别

**关键词**：不释放锁 | 释放锁 | 唤醒 | 同步 | 超时

**核心区别**：
```
              sleep()            wait()
释放锁        ❌ 不释放           ✓ 释放
必须同步      否                 是（synchronized）
唤醒方式      时间到             notify() / notifyAll()
异常          InterruptedException InterruptedException
位置          任何地方           对象内
超时          支持               支持
定时          System.currentTimeMillis() 配合使用
```

**使用场景**：
```
sleep() → 延迟执行、定时任务
wait()  → 线程间同步、生产者消费者
```

---

### 1️⃣4️⃣ 悲观锁和乐观锁

**关键词**：互斥 | CAS | 竞争 | 性能 | 应用场景

**核心区别**：
```
          悲观锁           乐观锁
假设      数据会冲突       数据不冲突
实现      synchronized    CAS（Compare And Swap）
性能      低（竞争高时）   高（竞争低时）
场景      竞争激烈        竞争不激烈
成本      锁成本           重试成本
```

**应用**：
```
悲观锁 → synchronized、Lock
乐观锁 → Atomic 原子类、CAS 操作
```

---

### 1️⃣5️⃣ BlockingQueue 的原理和使用场景

**关键词**：阻塞 | 生产消费 | 线程安全 | 队列 | 场景

**核心特点**：
- 元素满时阻塞添加
- 元素空时阻塞取出
- 线程安全

**常见实现**：
```
ArrayBlockingQueue  → 数组实现，有界
LinkedBlockingQueue → 链表实现，可无界
PriorityBlockingQueue → 优先级队列
DelayQueue          → 延时队列
SynchronousQueue    → 直接传递
```

**使用场景**：
```
1. 生产者消费者模式
2. 线程池任务队列
3. 消息队列
4. 流量控制
```

---

### 1️⃣6️⃣ 线程安全的集合

**关键词**：同步 | Concurrent | Copy-On-Write | 性能 | 选择

**线程安全集合**：
```
List:
- Vector（遗留）
- Collections.synchronizedList()
- CopyOnWriteArrayList（读多写少）

Map:
- Hashtable（遗留）
- Collections.synchronizedMap()
- ConcurrentHashMap（高效）

Set:
- Collections.synchronizedSet()
- CopyOnWriteArraySet

Queue:
- ConcurrentLinkedQueue
- BlockingQueue（阻塞）
```

**选择建议**：
```
- 多读少写 → CopyOnWriteArrayList
- 多读多写 → ConcurrentHashMap
- 队列 → ConcurrentLinkedQueue 或 BlockingQueue
- 避免 Vector / Hashtable（性能差）
```

---

### 1️⃣7️⃣ Atomic 原子类的原理

**关键词**：CAS | Compare And Swap | 无锁 | 原子操作 | 自旋

**核心原理**：
```
CAS(期望值, 新值)
─ 比较当前值与期望值
─ 相同 → 更新为新值
─ 不同 → 自旋重试

无锁 → 高性能
```

**常见类**：
```
AtomicInteger   → int 型原子操作
AtomicLong      → long 型原子操作
AtomicReference → 引用型原子操作
AtomicIntegerArray → int 数组原子操作
```

**缺点**：
```
- 自旋导致 CPU 消耗（高竞争时）
- 只适合简单操作
- 不能替代同步（无法保证操作序列原子性）
```

---

### 1️⃣8️⃣ ThreadLocal 的使用场景

**关键词**：线程隔离 | 上下文 | 数据库连接 | 用户信息 | 请求追踪

**核心应用**：
```
1. 数据库连接管理
   Connection conn = ConnectionPool.getConnection()

2. 用户会话信息
   User user = UserContext.getCurrentUser()

3. 事务管理
   Transaction tx = TransactionManager.getTransaction()

4. 请求 ID 追踪
   String requestId = RequestContext.getRequestId()

5. SimpleDateFormat 线程安全
   DateFormat df = threadLocal.get()
```

**注意**：
```
⚠️ 使用完毕必须 remove()
✓ 配合 try-finally 使用
✓ 避免内存泄漏
```

---

---

## 📌 Java 虚拟机部分

### 1️⃣ Java 垃圾回收机制

**关键词**：GC | 堆内存 | 回收标记 | 分代 | 算法

**GC 触发时机**：
```
- Eden 区满（Minor GC）
- Old 区满（Full GC）
- 显式调用 System.gc()
```

**回收算法**：
```
1. 标记-清除：标记存活→清除死对象（产生碎片）
2. 标记-整理：标记存活→移动→清除（无碎片）
3. 复制：分为 From/To，复制存活对象（双倍内存）
4. 分代：分 Young 代和 Old 代（最常用）
```

**分代垃圾回收**：
```
Young Gen（年轻代，GC 频繁）
├── Eden（新对象）→ Minor GC → Survivor S0/S1
├── Survivor S0/S1（存活对象）
└── → 经历 N 次 GC 后 → Old Gen

Old Gen（老年代，GC 不频繁）
└── → Full GC
```

---

### 2️⃣ 强、软、弱、虚引用

**关键词**：引用强度 | GC 回收 | 缓存 | WeakHashMap | 幻象

**四种引用**：
```
强引用（StrongReference）
- 普通引用：Object obj = new Object()
- GC 不会回收（除非无引用）
- 用途：普通对象

软引用（SoftReference）
- 内存不足时回收
- 用途：缓存（LRU 缓存）

弱引用（WeakReference）
- GC 就会回收
- 用途：避免内存泄漏（如 ThreadLocalMap）

虚引用（PhantomReference）
- 形同虚设，随时可回收
- 用途：跟踪对象被回收，用于资源释放
```

**强度排序**：强 > 软 > 弱 > 虚

**应用场景**：
```
软引用 → ImageCache = new SoftReference<Image>()
弱引用 → WeakHashMap<Key, Value>()
虚引用 → PhantomReference 监听对象回收
```

---

### 3️⃣ JVM 类加载机制

**关键词**：双亲委派 | 加载 | 验证 | 初始化 | 类加载器

**类加载过程**：
```
1. 加载（Loading）
   → 获取类的二进制字节流
   → 创建代表这个类的 java.lang.Class 对象

2. 验证（Verification）
   → 文件格式验证
   → 元数据验证
   → 字节码验证
   → 符号引用验证

3. 准备（Preparation）
   → 分配内存给类变量
   → 赋默认初始值（0、false、null）

4. 解析（Resolution）
   → 符号引用 → 直接引用

5. 初始化（Initialization）
   → 执行静态初始化块
   → 静态变量赋值
```

**双亲委派**：
```
AppClassLoader
    ↓ 委派
PlatformClassLoader
    ↓ 委派
BootstrapClassLoader
```

**优势**：
- 避免类重复加载
- 保证类的唯一性（同一 ClassLoader 加载）

---

### 4️⃣ JVM、Dalvik、ART 三者

**关键词**：字节码 | Dex | 即时编译 | 预编译 | Android

**核心对比**：
```
          JVM          Dalvik       ART
字节码    .class       .dex         .dex
执行      解释 + JIT   解释         AOT（预编译）
性能      中等         低           高
编译      JIT          Just-in-time AOT（Ahead-of-time）
内存      较少         中等         较多
启动      快           一般         慢（首次）
```

**发展历程**：
```
JVM（Java Virtual Machine）
    ↓
Dalvik（Android 4.4 前）
    ↓
ART（Android 5.0+）
```

---

### 5️⃣ Java 的内存回收机制

**关键词**：标记 | 清除 | GC Root | 引用链 | 堆 | 栈

**内存回收步骤**：
```
1. 标记存活对象
   - 从 GC Root 开始
   - 标记所有可达对象

2. 清除死对象
   - 清除未标记对象
   - 回收内存

3. 可选整理
   - 移动存活对象
   - 消除内存碎片
```

**GC Root**：
```
- 正在执行的线程栈帧中的局部变量
- 类的静态变量
- 常量池中的引用
- JNI 引用的对象
```

---

### 6️⃣ JMM（Java Memory Model）

**关键词**：可见性 | 有序性 | happens-before | volatile | synchronized

**核心**：
```
JMM 保证：
1. 可见性：一个线程的修改对另一线程可见
2. 有序性：指令不会被乱序执行
3. 原子性：某些操作不会被中断
```

**happens-before 原则**：
```
1. 单线程：前面的操作 happens-before 后面的操作
2. volatile：写 happens-before 读
3. synchronized：解锁 happens-before 加锁
4. Thread.start() happens-before 线程内任何操作
5. 传递性：A hb B，B hb C → A hb C
```

**问题**：
```
- 指令重排
- 缓存不一致
- 编译器优化导致乱序

解决**：
✓ volatile（防止重排，保证可见性）
✓ synchronized（互斥）
✓ Lock（互斥，更灵活）
✓ Atomic（CAS）
```

---

---

## 📊 学习进度追踪

### 学习建议
1. **第1天**：快速浏览所有关键词，了解知识框架
2. **第2-3天**：每天深入学习 5-10 道题目（对照完整笔记）
3. **第4-7天**：用快速复习卡回顾前面学过的题目
4. **第8天+**：用 Anki 卡片进行间隔重复

### 背诵技巧
- 📌 **关键词记忆法**：先记住 3-5 个关键词，再推导完整答案
- 🎯 **对比记忆法**：通过表格对比异同，加深印象
- 📝 **代码记忆法**：手写代码示例，培养肌肉记忆
- 🔄 **间隔重复法**：间隔 1 天、3 天、7 天、14 天复习

---

**下一步**：结合《Java_Complete_Study_Notes.md》进行深度学习！
