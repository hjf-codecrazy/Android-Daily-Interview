# Java 完整学习笔记

> 包含详细答案 + 代码示例，适合深度学习和理解

---

## 📌 Java 基础部分

### 1. 抽象类与接口的区别？

#### 详细答案

| 特性 | 抽象类 | 接口 |
|------|--------|------|
| 声明关键字 | `abstract class` | `interface` |
| 成员变量 | 任意修饰符(public/private/protected) | 默认 `public static final` |
| 方法 | 可有抽象和具体方法 | Java8前只能有抽象方法，Java8+可有默认方法 |
| 继承/实现 | 单继承 `extends` | 多实现 `implements` |
| 初始化方式 | 有构造方法 | 没有构造方法 |
| 访问修饰符 | 可使用任意修饰符 | 只能 public 和 protected |
| 设计目的 | 表示"是一个(IS-A)" | 表示"能够做(CAN-DO)" |

#### 代码示例

```java
// 抽象类示例
abstract class Animal {
    // 具体变量
    protected String name;

    // 构造方法
    public Animal(String name) {
        this.name = name;
    }

    // 具体方法
    public void eat() {
        System.out.println(name + " is eating");
    }

    // 抽象方法
    abstract void makeSound();
}

class Dog extends Animal {
    public Dog(String name) {
        super(name);
    }

    @Override
    void makeSound() {
        System.out.println("Woof!");
    }
}

// 接口示例
interface Flyable {
    // 默认是 public static final
    int MAX_HEIGHT = 10000;

    // 抽象方法
    void fly();

    // Java8 默认方法
    default void land() {
        System.out.println("Landing...");
    }

    // Java9 私有方法
    private void prepareForFlight() {
        System.out.println("Preparing...");
    }
}

class Bird implements Flyable {
    @Override
    public void fly() {
        System.out.println("Flying high!");
    }
}

// 多实现
class Airplane implements Flyable, Comparable<Airplane> {
    @Override
    public void fly() {
        System.out.println("Airplane flying");
    }

    @Override
    public int compareTo(Airplane o) {
        return 0;
    }
}
```

---

### 2. final、static 和 synchronized 的修饰范围和作用

#### 详细答案

**final 关键字**

| 修饰对象 | 作用 | 示例 |
|---------|------|------|
| 类 | 该类不能被继承 | `final class ImmutableClass {}` |
| 方法 | 方法不能被重写 | `final void method() {}` |
| 变量 | 变量只能赋值一次 | `final int VAL = 10;` |

**static 关键字**

| 修饰对象 | 作用 | 示例 |
|---------|------|------|
| 变量 | 静态变量，属于类而非对象，全局共享 | `static int count = 0;` |
| 方法 | 静态方法，可通过类名直接调用 | `static void method() {}` |
| 代码块 | 类加载时执行一次，优先级高 | `static { init(); }` |
| 内部类 | 静态内部类，可不依赖外部类实例 | `static class Inner {}` |

**synchronized 关键字**

| 修饰对象 | 作用 | 示例 |
|---------|------|------|
| 方法 | 同步方法，锁定当前对象(this) | `synchronized void method() {}` |
| 代码块 | 同步块，可指定锁对象 | `synchronized(obj) { ... }` |
| 静态方法 | 锁定类对象(Class) | `synchronized static void method() {}` |

#### 代码示例

```java
// final 示例
final class Person {
    private final String name;  // 不可改变
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public final void display() {  // 不能被重写
        System.out.println(name + ": " + age);
    }
}

// static 示例
class Counter {
    static int count = 0;  // 共享变量

    static {  // 静态初始化块
        System.out.println("Counter class loaded");
        count = 100;
    }

    static void increment() {  // 静态方法
        count++;
    }
}

// synchronized 示例
class ThreadSafeCounter {
    private int count = 0;

    // 同步方法
    public synchronized void increment() {
        count++;
    }

    // 同步代码块
    public void incrementEfficient() {
        synchronized(this) {
            count++;
        }
    }

    // 静态同步方法，锁定类
    public static synchronized void staticIncrement() {
        // 操作静态变量
    }
}
```

---

### 3. String、StringBuffer 和 StringBuilder 的区别

#### 详细答案

| 特性 | String | StringBuffer | StringBuilder |
|------|--------|-------------|--------------|
| 可变性 | **不可变** | **可变** | **可变** |
| 线程安全 | 线程安全(不可变) | 线程安全(synchronized) | 线程不安全 |
| 性能 | 低(频繁创建新对象) | 中等 | **高** |
| 使用场景 | 不需要修改 | 多线程修改字符串 | 单线程修改字符串 |
| 继承关系 | 继承Object | 继承AbstractStringBuilder | 继承AbstractStringBuilder |

#### 代码示例

```java
// String 示例
String str = "Hello";
str = str + " World";  // 创建新对象
String str2 = new String("Hello");  // 创建两个对象

// 内存中实际上有多个对象：
// "Hello" -> "Hello World"（新对象）

// StringBuffer 示例（线程安全）
StringBuffer sb = new StringBuffer("Hello");
sb.append(" World");  // 修改原对象
sb.insert(0, "Say ");  // 在指定位置插入
sb.reverse();  // 反转
System.out.println(sb);  // dlorW olleH yaS

// StringBuilder 示例（性能更好）
StringBuilder sb2 = new StringBuilder("Hello");
sb2.append(" World");
sb2.insert(0, "Say ");
System.out.println(sb2);

// 性能对比
long start = System.currentTimeMillis();

// 方式1：String 拼接（最慢）
String result = "";
for(int i = 0; i < 100000; i++) {
    result += i;  // 每次都创建新对象
}

// 方式2：StringBuffer（中等）
StringBuffer sb3 = new StringBuffer();
for(int i = 0; i < 100000; i++) {
    sb3.append(i);
}
String result2 = sb3.toString();

// 方式3：StringBuilder（最快）
StringBuilder sb4 = new StringBuilder();
for(int i = 0; i < 100000; i++) {
    sb4.append(i);
}
String result3 = sb4.toString();

long end = System.currentTimeMillis();
System.out.println("耗时: " + (end - start) + "ms");
```

---

### 4. equals()、== 和 hashCode() 的区别和使用场景

#### 详细答案

| 比较项 | == | equals() | hashCode() |
|--------|-----|----------|-----------|
| 作用 | 比较引用是否相同 | 比较对象内容是否相同 | 获取哈希值 |
| 比较对象 | 基本类型/引用 | 对象内容 | 对象哈希值 |
| 默认实现 | 比较地址 | 默认同==,可重写 | 对象内存地址转哈希值 |
| 性能 | **最快** | 较慢 | 中等 |
| 用途 | 引用相同判断 | 对象相等判断 | HashMap/HashSet key |

#### 代码示例

```java
public class Student {
    private String name;
    private int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // 重写 equals 方法
    @Override
    public boolean equals(Object obj) {
        // 1. 引用相同
        if (this == obj) {
            return true;
        }

        // 2. 对象为null或类型不同
        if (obj == null || !(obj instanceof Student)) {
            return false;
        }

        // 3. 比较内容
        Student other = (Student) obj;
        return this.name.equals(other.name) && this.age == other.age;
    }

    // 重写 hashCode 方法
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
        // 或者手动计算：
        // return name.hashCode() * 31 + age;
    }
}

// 使用示例
public class Main {
    public static void main(String[] args) {
        Student s1 = new Student("Tom", 20);
        Student s2 = new Student("Tom", 20);
        Student s3 = s1;

        // == 比较引用
        System.out.println(s1 == s2);   // false
        System.out.println(s1 == s3);   // true

        // equals 比较内容
        System.out.println(s1.equals(s2));  // true
        System.out.println(s1.equals(s3));  // true

        // hashCode
        System.out.println(s1.hashCode());  // 哈希值
        System.out.println(s2.hashCode());  // 相同哈希值

        // HashMap 中的应用
        Map<Student, String> map = new HashMap<>();
        map.put(s1, "Student1");
        System.out.println(map.get(s2));  // 可以找到，因为equals返回true

        // HashSet 中的应用
        Set<Student> set = new HashSet<>();
        set.add(s1);
        set.add(s2);  // 不会重复添加，因为equals为true
        System.out.println(set.size());  // 1
    }
}

// 为什么要同时重写 equals 和 hashCode？
// 原因：保证 hashCode 相等的两个对象，equals 也相等
// 如果只重写 equals，HashMap/HashSet 中会出现问题
```

---

### 5. Java 中深拷贝与浅拷贝的区别

#### 详细答案

| 特性 | 浅拷贝 | 深拷贝 |
|------|--------|--------|
| 对象本身 | 复制 | 复制 |
| 对象属性(基本类型) | 复制值 | 复制值 |
| 对象属性(引用类型) | 复制引用 | 复制对象内容 |
| 内存占用 | 较少 | 较多 |
| 修改影响 | 会相互影响 | 不相互影响 |

#### 代码示例

```java
// 被拷贝的对象
class Address {
    public String city;

    public Address(String city) {
        this.city = city;
    }
}

class Person implements Cloneable {
    public String name;
    public Address address;

    public Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    // 浅拷贝
    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();  // Object 的 clone 是浅拷贝
    }

    // 深拷贝方式1：手动复制
    public Person deepCopyManual() {
        Person person = new Person(this.name, new Address(this.address.city));
        return person;
    }

    // 深拷贝方式2：通过序列化
    public Person deepCopySerialization() throws Exception {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(this);

        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        return (Person) ois.readObject();
    }
}

// 测试
public class CloneTest {
    public static void main(String[] args) throws Exception {
        Person p1 = new Person("Tom", new Address("Beijing"));

        // 浅拷贝
        Person p2 = (Person) p1.clone();
        p2.name = "Jerry";
        p2.address.city = "Shanghai";

        System.out.println(p1.name);           // Tom
        System.out.println(p1.address.city);   // Shanghai (被改变了！)
        System.out.println(p2.name);           // Jerry
        System.out.println(p2.address.city);   // Shanghai

        // 深拷贝
        Person p3 = new Person("Tom", new Address("Beijing"));
        Person p4 = p3.deepCopyManual();
        p4.name = "Jerry";
        p4.address.city = "Shanghai";

        System.out.println(p3.name);           // Tom
        System.out.println(p3.address.city);   // Beijing (不变)
        System.out.println(p4.name);           // Jerry
        System.out.println(p4.address.city);   // Shanghai
    }
}
```

---

### 6. Error 和 Exception 的区别

#### 详细答案

| 特性 | Error | Exception |
|------|-------|----------|
| 继承关系 | 继承 Throwable | 继承 Throwable |
| 恢复可能性 | **不可恢复** | **可恢复** |
| 捕获方式 | 通常不捕获 | 应该捕获处理 |
| 常见类型 | StackOverflowError、OutOfMemoryError | IOException、SQLException |
| 检查异常 | 都是运行时异常 | 有检查异常和运行时异常 |

```
Throwable
├── Error（系统错误，不可恢复）
│   ├── StackOverflowError
│   ├── OutOfMemoryError
│   └── VirtualMachineError
└── Exception（可恢复异常）
    ├── CheckedException（检查异常，必须处理）
    │   ├── IOException
    │   ├── SQLException
    │   └── ClassNotFoundException
    └── RuntimeException（运行时异常，可不处理）
        ├── NullPointerException
        ├── ArrayIndexOutOfBoundsException
        ├── ClassCastException
        └── ArithmeticException
```

#### 代码示例

```java
// Error 示例
public class ErrorDemo {
    public static void main(String[] args) {
        try {
            stackOverflow();  // StackOverflowError
        } catch (StackOverflowError e) {
            System.out.println("StackOverflow: " + e.getMessage());
        }
    }

    public static void stackOverflow() {
        stackOverflow();  // 无限递归
    }
}

// Exception 示例
public class ExceptionDemo {
    public static void main(String[] args) {
        // 检查异常（Checked Exception）必须处理
        try {
            FileInputStream fis = new FileInputStream("test.txt");
            fis.close();
        } catch (FileNotFoundException e) {
            System.out.println("File not found: " + e.getMessage());
        } catch (IOException e) {
            System.out.println("IO error: " + e.getMessage());
        }

        // 运行时异常（Runtime Exception）可选处理
        try {
            int[] arr = new int[5];
            System.out.println(arr[10]);  // ArrayIndexOutOfBoundsException
        } catch (ArrayIndexOutOfBoundsException e) {
            System.out.println("Array index error: " + e.getMessage());
        }

        // NullPointerException
        try {
            String str = null;
            System.out.println(str.length());  // NullPointerException
        } catch (NullPointerException e) {
            System.out.println("Null pointer: " + e.getMessage());
        }
    }
}

// 自定义异常
class AgeException extends Exception {
    public AgeException(String message) {
        super(message);
    }
}

class Person {
    private int age;

    public void setAge(int age) throws AgeException {
        if (age < 0 || age > 150) {
            throw new AgeException("Age must be between 0 and 150");
        }
        this.age = age;
    }
}
```

---

### 7. 什么是反射机制？反射机制的应用场景

#### 详细答案

反射是 Java 的一个强大特性，允许程序在运行时动态获取类的信息和调用方法。

**获取 Class 对象的三种方式：**

```
1. Class.forName("类的全限定名")
2. ClassName.class
3. instance.getClass()
```

#### 代码示例

```java
public class ReflectionDemo {

    public static void main(String[] args) throws Exception {
        // 获取 Class 对象
        Class<?> clazz = Class.forName("java.lang.String");
        // 或者
        Class<String> clazz2 = String.class;
        // 或者
        String str = "test";
        Class<?> clazz3 = str.getClass();

        // 1. 获取类信息
        System.out.println("类名: " + clazz.getName());
        System.out.println("简单名: " + clazz.getSimpleName());
        System.out.println("包名: " + clazz.getPackage().getName());

        // 2. 获取构造器
        Constructor<?>[] constructors = clazz.getConstructors();
        for (Constructor<?> c : constructors) {
            System.out.println("构造器: " + c);
        }

        // 3. 获取字段
        Field[] fields = clazz.getDeclaredFields();
        for (Field f : fields) {
            System.out.println("字段: " + f.getName() + ", 类型: " + f.getType());
        }

        // 4. 获取方法
        Method[] methods = clazz.getDeclaredMethods();
        for (Method m : methods) {
            System.out.println("方法: " + m.getName());
        }

        // 5. 创建对象
        Object instance = clazz.newInstance();

        // 6. 调用方法
        Method method = clazz.getDeclaredMethod("length");
        Object result = method.invoke(str);
        System.out.println("字符串长度: " + result);

        // 7. 访问和修改字段
        Class<?> personClass = Class.forName("Person");
        Object person = personClass.newInstance();
        Field nameField = personClass.getDeclaredField("name");
        nameField.setAccessible(true);  // 跳过访问权限检查
        nameField.set(person, "Tom");
        System.out.println("名字: " + nameField.get(person));
    }
}

// 自定义类
class Person {
    private String name;
    private int age;

    public Person() {}

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void display() {
        System.out.println(name + ": " + age);
    }
}

// 反射应用场景：
// 1. 框架层面的应用（Spring、Hibernate）
// 2. 序列化和反序列化（JSON解析）
// 3. 动态代理
// 4. 注解处理
// 5. ORM 映射
// 6. 插件系统
```

---

### 8. 如何重写 equals() 方法？为什么还要重写 hashCode()？

#### 详细答案

**重写 equals() 的规则：**
1. 自反性：`a.equals(a)` 必须为 true
2. 对称性：`a.equals(b)` 则 `b.equals(a)`
3. 传递性：`a.equals(b)` 且 `b.equals(c)` 则 `a.equals(c)`
4. 一致性：多次调用结果相同
5. 与 null 的比较：`a.equals(null)` 必须为 false

**为什么重写 equals() 要重写 hashCode()？**
- HashMap/HashSet 先用 hashCode() 定位，再用 equals() 判断相等
- 如果 `equals()` 为 true，但 `hashCode()` 不同，会导致数据错误
- HashCode 契约：**equals 相等 → hashCode 必须相等**（反之不必成立）

#### 代码示例

```java
public class Employee {
    private String id;
    private String name;

    public Employee(String id, String name) {
        this.id = id;
        this.name = name;
    }

    // 重写 equals
    @Override
    public boolean equals(Object obj) {
        // 1. 检查引用相同
        if (this == obj) return true;

        // 2. 检查 null 和类型
        if (obj == null || !(obj instanceof Employee)) return false;

        // 3. 强制转换
        Employee other = (Employee) obj;

        // 4. 比较关键字段
        return this.id.equals(other.id) && this.name.equals(other.name);
    }

    // 必须同时重写 hashCode
    @Override
    public int hashCode() {
        return Objects.hash(id, name);
        // 或手动计算：
        // return id.hashCode() * 31 + name.hashCode();
    }
}

// 测试
public class HashTest {
    public static void main(String[] args) {
        Employee e1 = new Employee("001", "Tom");
        Employee e2 = new Employee("001", "Tom");

        System.out.println(e1.equals(e2));      // true
        System.out.println(e1.hashCode() == e2.hashCode());  // true

        // HashMap 正确使用
        Map<Employee, String> map = new HashMap<>();
        map.put(e1, "Employee1");
        System.out.println(map.get(e2));       // 可以获取（因为hashCode和equals都匹配）

        // HashSet 正确使用
        Set<Employee> set = new HashSet<>();
        set.add(e1);
        set.add(e2);  // 不会重复添加
        System.out.println(set.size());         // 1
    }
}

// 常见错误案例（只重写equals，不重写hashCode）
class BadEmployee {
    private String id;

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof BadEmployee)) return false;
        return this.id.equals(((BadEmployee) obj).id);
    }
    // 没有重写 hashCode，会导致 HashMap 失效
}
```

---

### 9. Java 中 IO 流分为几种？它们之间有什么区别？

#### 详细答案

**按数据流向分类：**
- 输入流（Input Stream）：从源读取数据
- 输出流（Output Stream）：向目标写入数据

**按处理数据单位分类：**
- 字节流：处理单个字节（8 bits），适合二进制数据
- 字符流：处理字符（16 bits），适合文本数据

**按功能分类：**
- 节点流：直接操作数据源
- 处理流：对节点流进行包装

```
InputStream (字节输入流)
├── FileInputStream
├── ByteArrayInputStream
├── ObjectInputStream
└── FilterInputStream
    ├── BufferedInputStream
    └── DataInputStream

OutputStream (字节输出流)
├── FileOutputStream
├── ByteArrayOutputStream
├── ObjectOutputStream
└── FilterOutputStream
    ├── BufferedOutputStream
    └── DataOutputStream

Reader (字符输入流)
├── FileReader
├── StringReader
└── BufferedReader

Writer (字符输出流)
├── FileWriter
├── StringWriter
└── BufferedWriter
```

#### 代码示例

```java
public class IOStreamDemo {

    // 1. 字节流读写
    public static void byteStreamDemo() throws IOException {
        // 写入
        try (FileOutputStream fos = new FileOutputStream("test.bin");
             BufferedOutputStream bos = new BufferedOutputStream(fos)) {
            bos.write("Hello".getBytes());
            bos.flush();
        }

        // 读取
        try (FileInputStream fis = new FileInputStream("test.bin");
             BufferedInputStream bis = new BufferedInputStream(fis)) {
            byte[] buffer = new byte[1024];
            int length = bis.read(buffer);
            System.out.println(new String(buffer, 0, length));
        }
    }

    // 2. 字符流读写（文本数据）
    public static void charStreamDemo() throws IOException {
        // 写入
        try (FileWriter fw = new FileWriter("test.txt");
             BufferedWriter bw = new BufferedWriter(fw)) {
            bw.write("Hello World\n");
            bw.write("Java IO");
            bw.flush();
        }

        // 读取
        try (FileReader fr = new FileReader("test.txt");
             BufferedReader br = new BufferedReader(fr)) {
            String line;
            while ((line = br.readLine()) != null) {
                System.out.println(line);
            }
        }
    }

    // 3. 对象序列化
    public static void objectStreamDemo() throws Exception {
        // 写入对象
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("object.dat"))) {
            oos.writeObject(new Person("Tom", 25));
        }

        // 读取对象
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("object.dat"))) {
            Person person = (Person) ois.readObject();
            System.out.println(person);
        }
    }

    // 4. 字节数组流（内存操作）
    public static void byteArrayStreamDemo() throws IOException {
        // 写入内存
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        baos.write("Hello".getBytes());
        baos.write("World".getBytes());
        byte[] data = baos.toByteArray();

        // 从内存读取
        ByteArrayInputStream bais = new ByteArrayInputStream(data);
        byte[] buffer = new byte[5];
        bais.read(buffer);
        System.out.println(new String(buffer));
    }
}

class Person implements Serializable {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
}

// IO 流性能对比
public class IOPerformance {
    public static void main(String[] args) throws IOException {
        long start, end;
        String file = "largefile.txt";

        // 方式1：直接字节流（慢）
        start = System.currentTimeMillis();
        try (FileInputStream fis = new FileInputStream(file)) {
            int b;
            while ((b = fis.read()) != -1) {
                // 处理
            }
        }
        end = System.currentTimeMillis();
        System.out.println("字节流: " + (end - start) + "ms");

        // 方式2：缓冲字节流（快）
        start = System.currentTimeMillis();
        try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream(file))) {
            byte[] buffer = new byte[8192];
            while (bis.read(buffer) != -1) {
                // 处理
            }
        }
        end = System.currentTimeMillis();
        System.out.println("缓冲流: " + (end - start) + "ms");
    }
}
```

---

### 10. Java 泛型中的类型擦除及其局限性

#### 详细答案

**类型擦除：**
Java 泛型在编译时进行类型检查，但在运行时会被擦除为 Object 类型或其边界类型。这是为了保持与旧版本 Java 的兼容性。

**类型擦除的三个阶段：**
1. 替换：用边界类型（默认 Object）替换类型参数
2. 移除：移除类型参数声明
3. 插入：插入类型强制转换

#### 代码示例

```java
public class GenericErasureDemo {

    // 原始代码
    public static <T> T getValue(T t) {
        return t;
    }

    // 编译后（类型擦除）
    public static Object getValueAfterErasure(Object t) {
        return t;
    }

    // 带有上界的泛型
    public static <T extends Number> double sum(T a, T b) {
        return a.doubleValue() + b.doubleValue();
    }

    // 编译后（用 Number 替换 T）
    public static double sumAfterErasure(Number a, Number b) {
        return a.doubleValue() + b.doubleValue();
    }

    public static void main(String[] args) {
        // 1. 类型擦除导致的问题
        List<String> list1 = new ArrayList<>();
        List<Integer> list2 = new ArrayList<>();

        // 运行时，两个 list 类型相同
        System.out.println(list1.getClass() == list2.getClass());  // true
        System.out.println(list1.getClass());  // class java.util.ArrayList

        // 2. 无法创建泛型数组
        // List<String>[] arrays = new List<String>[10];  // 编译错误

        // 解决方案：使用通配符或 ArrayList
        List<?>[] arrays = new List[10];
        arrays[0] = new ArrayList<String>();

        // 3. 无法直接通过 instanceof 检查泛型类型
        List<String> strings = new ArrayList<>();
        // if (strings instanceof List<String>) {}  // 编译错误
        // 可以这样：
        if (strings instanceof List) {}  // 正确

        // 4. 无法通过反射获取泛型参数的具体类型
        Method method = null;
        try {
            method = GenericErasureDemo.class.getMethod("getValue", Object.class);
            System.out.println(method.getGenericReturnType());  // T
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }
}

// 使用继承保存泛型信息的方法
public class GenericTypeToken<T> {
    private Type type;

    public GenericTypeToken() {
        // 获取泛型父类
        Type superClass = this.getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
    }

    public Type getType() {
        return type;
    }
}

// 使用示例
public class TypeTokenExample extends GenericTypeToken<String> {
    public static void main(String[] args) {
        TypeTokenExample example = new TypeTokenExample();
        System.out.println(example.getType());  // class java.lang.String
    }
}

// 类型擦除的局限性总结：
// 1. 不能创建泛型数组
// 2. 不能创建泛型异常类
// 3. 不能通过 instanceof 检查泛型类型
// 4. 不能通过反射获取泛型参数的运行时类型
// 5. 不能定义泛型静态变量
// 6. 泛型类的所有实例在运行时是同一个类
```

---

### 11. String 为什么要设计成不可变的？

#### 详细答案

**String 不可变的设计原因：**

1. **安全性**：字符串常被用作 HashMap/HashSet 的 key 和数据库连接参数，不可变保证在使用过程中不会被修改

2. **性能**：字符串对象可以被缓存和重用，减少内存占用

3. **多线程安全**：不可变对象天生线程安全

4. **哈希值缓存**：不可变的 String 可以缓存其哈希值，加快 HashMap 查询速度

#### 代码示例

```java
public class StringImmutableDemo {

    public static void main(String[] args) {
        // 1. String 对象的不可变性演示
        String str1 = "Hello";
        String str2 = str1.concat(" World");

        System.out.println(str1);  // Hello (未改变)
        System.out.println(str2);  // Hello World (新对象)
        System.out.println(str1 == str2);  // false (不同对象)

        // 2. 字符串池（String Pool）
        String s1 = "Java";
        String s2 = "Java";
        String s3 = new String("Java");

        System.out.println(s1 == s2);      // true (同一对象)
        System.out.println(s1 == s3);      // false (不同对象)
        System.out.println(s1.equals(s3)); // true (内容相同)

        // 3. HashMap 中的安全性
        Map<String, Integer> map = new HashMap<>();
        String key = new String("name");
        map.put(key, 100);

        // 如果 String 可变，修改 key 会导致查询失败
        // key.toUpperCase();  // 不会改变原字符串

        System.out.println(map.get("name"));  // 仍能找到

        // 4. 多线程安全
        // 不可变的 String 可以安全地在多个线程间共享
        String shared = "Hello";  // 可以直接在多线程中使用，不需要同步

        // 5. 哈希值缓存
        String str = "test";
        int hash1 = str.hashCode();
        int hash2 = str.hashCode();  // 直接返回缓存的哈希值，无需重新计算
        System.out.println(hash1 == hash2);  // true
    }
}

// String 不可变的实现原理
public class StringImplementation {
    // String 源代码（简化版本）
    public static class StringDemo {
        // 1. 字符数组被 private final 修饰
        private final char[] value;

        // 2. 哈希值被缓存
        private int hash;

        public StringDemo(char[] value) {
            // 3. 防御性复制（避免外部修改传入的数组）
            this.value = value.clone();
        }

        // 4. 没有提供修改 value 的方法
        public char charAt(int index) {
            return value[index];
        }

        // 5. 返回新的 String 而不是修改原对象
        public String concat(String str) {
            // 返回新的 String 对象
            return new String(/* ... */);
        }

        @Override
        public int hashCode() {
            // 6. 缓存哈希值
            int h = hash;
            if (h == 0 && value.length > 0) {
                for (char c : value) {
                    h = 31 * h + c;
                }
                hash = h;
            }
            return h;
        }
    }
}

// 不可变性的好处
public class ImmutableBenefits {

    // 安全地作为 HashMap key
    public static void hashMapSafety() {
        Map<String, String> map = new HashMap<>();
        String key = "user_id";
        map.put(key, "123");

        // 即使外部代码持有对 key 的引用，也不必担心被修改
        useKey(key);
        System.out.println(map.get(key));  // 仍能获取
    }

    private static void useKey(String key) {
        // 无法修改 key
        // key 不可能被修改
    }

    // 线程安全的共享
    public static void threadSafety() {
        String shared = "GlobalData";

        // 可以安全地在多个线程间传递，无需同步
        Thread t1 = new Thread(() -> {
            System.out.println(shared);
        });

        Thread t2 = new Thread(() -> {
            System.out.println(shared);
        });
    }
}
```

---

### 12. Java 注解的理解

#### 详细答案

**注解的三个层次：**
1. **编译时注解**：只在编译时有效（如 @Override）
2. **加载时注解**：在类加载时处理
3. **运行时注解**：在程序运行时处理（需要 @Retention(RetentionPolicy.RUNTIME)）

**常用的元注解：**
- `@Retention`：指定注解的生命周期
- `@Target`：指定注解用在哪里
- `@Inherited`：指定注解是否被继承
- `@Documented`：指定是否被 JavaDoc 文档化

#### 代码示例

```java
import java.lang.annotation.*;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

// 1. 定义自定义注解
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface MyAnnotation {
    String value() default "default value";
    int age() default 0;
}

// 2. 定义字段注解
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyFieldAnnotation {
    String columnName();
    int length() default 255;
}

// 3. 使用注解
@MyAnnotation(value = "Class Annotation", age = 10)
class AnnotatedClass {
    @MyFieldAnnotation(columnName = "user_name", length = 50)
    private String name;

    @MyFieldAnnotation(columnName = "user_age")
    private int age;

    @MyAnnotation("Method Annotation")
    public void display() {
        System.out.println(name + ": " + age);
    }
}

// 4. 处理注解
public class AnnotationProcessor {

    public static void main(String[] args) {
        // 反射获取并处理注解
        Class<?> clazz = AnnotatedClass.class;

        // 处理类注解
        if (clazz.isAnnotationPresent(MyAnnotation.class)) {
            MyAnnotation annotation = clazz.getAnnotation(MyAnnotation.class);
            System.out.println("类注解值: " + annotation.value());
            System.out.println("类注解年龄: " + annotation.age());
        }

        // 处理字段注解
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            if (field.isAnnotationPresent(MyFieldAnnotation.class)) {
                MyFieldAnnotation fieldAnnotation = field.getAnnotation(MyFieldAnnotation.class);
                System.out.println("字段: " + field.getName());
                System.out.println("列名: " + fieldAnnotation.columnName());
                System.out.println("长度: " + fieldAnnotation.length());
            }
        }

        // 处理方法注解
        try {
            Method method = clazz.getMethod("display");
            if (method.isAnnotationPresent(MyAnnotation.class)) {
                MyAnnotation methodAnnotation = method.getAnnotation(MyAnnotation.class);
                System.out.println("方法注解值: " + methodAnnotation.value());
            }
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    }
}

// 5. 实际应用：ORM 框架示例
@Target(ElementType.CLASS)
@Retention(RetentionPolicy.RUNTIME)
public @interface Entity {
    String tableName();
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    String name();
    boolean primaryKey() default false;
}

@Entity(tableName = "t_user")
class User {
    @Column(name = "id", primaryKey = true)
    private int id;

    @Column(name = "username")
    private String username;

    @Column(name = "email")
    private String email;
}

// ORM 框架的反射处理
public class ORMFramework {
    public static String generateSQL(Class<?> clazz) {
        Entity entity = clazz.getAnnotation(Entity.class);
        if (entity == null) return null;

        StringBuilder sql = new StringBuilder("CREATE TABLE " + entity.tableName() + " (");

        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            Column column = field.getAnnotation(Column.class);
            if (column != null) {
                sql.append(column.name()).append(" ");

                // 根据类型映射 SQL 类型
                if (field.getType() == int.class) {
                    sql.append("INT");
                } else if (field.getType() == String.class) {
                    sql.append("VARCHAR(255)");
                }

                if (column.primaryKey()) {
                    sql.append(" PRIMARY KEY");
                }

                sql.append(", ");
            }
        }

        sql.deleteCharAt(sql.length() - 1);
        sql.deleteCharAt(sql.length() - 1);
        sql.append(")");

        return sql.toString();
    }

    public static void main(String[] args) {
        String sql = generateSQL(User.class);
        System.out.println(sql);
        // CREATE TABLE t_user (id INT PRIMARY KEY, username VARCHAR(255), email VARCHAR(255))
    }
}

// 6. Java 内置注解
public class BuiltInAnnotations {
    @Override  // 编译时检查
    public String toString() {
        return "BuiltInAnnotations{}";
    }

    @Deprecated  // 标记已弃用
    public void oldMethod() {
    }

    @SuppressWarnings("unchecked")  // 抑制警告
    public void suppressWarning() {
        List list = new ArrayList();
    }

    @FunctionalInterface  // 标记函数式接口
    interface Converter {
        int convert(String str);
    }
}
```

---

### 13. Java 成员变量、局部变量和静态变量的创建和回收时机

#### 详细答案

| 变量类型 | 声明位置 | 创建时机 | 回收时机 | 初始值 | 作用域 |
|---------|---------|---------|---------|--------|--------|
| **成员变量** | 类体内 | 对象创建时 | 对象被GC回收 | 有(默认值) | 整个对象 |
| **局部变量** | 方法内 | 执行到该行时 | 方法结束 | 无(必须初始化) | 该方法 |
| **静态变量** | 类体内 | 类加载时 | 类被卸载时 | 有(默认值) | 整个程序 |

#### 代码示例

```java
public class VariableLifecycleDemo {

    // 1. 静态变量（类变量）
    static int staticVar = 0;  // 类加载时初始化
    static {
        staticVar = 100;  // 静态初始化块
        System.out.println("静态初始化块执行，staticVar = " + staticVar);
    }

    // 2. 成员变量（实例变量）
    int memberVar = 10;  // 对象创建时初始化
    {
        memberVar = 20;  // 非静态初始化块
        System.out.println("非静态初始化块执行，memberVar = " + memberVar);
    }

    public VariableLifecycleDemo() {
        memberVar = 30;  // 构造方法中
        System.out.println("构造方法执行，memberVar = " + memberVar);
    }

    public void method() {
        // 3. 局部变量（方法内）
        int localVar = 0;  // 执行到此行时创建
        System.out.println("局部变量 localVar = " + localVar);

        if (true) {
            int blockLocalVar = 50;  // 代码块内的局部变量
            System.out.println("块局部变量 blockLocalVar = " + blockLocalVar);
        }
        // blockLocalVar 不能访问
    }  // method 结束，localVar 被回收

    public static void main(String[] args) {
        System.out.println("=== 第一个对象 ===");
        VariableLifecycleDemo obj1 = new VariableLifecycleDemo();

        System.out.println("\n=== 第二个对象 ===");
        VariableLifecycleDemo obj2 = new VariableLifecycleDemo();

        System.out.println("\n=== 调用方法 ===");
        obj1.method();

        System.out.println("\nstaticVar = " + VariableLifecycleDemo.staticVar);
        System.out.println("obj1.memberVar = " + obj1.memberVar);
        System.out.println("obj2.memberVar = " + obj2.memberVar);
    }
}

// 输出：
// 静态初始化块执行，staticVar = 100
// === 第一个对象 ===
// 非静态初始化块执行，memberVar = 20
// 构造方法执行，memberVar = 30
//
// === 第二个对象 ===
// 非静态初始化块执行，memberVar = 20
// 构造方法执行，memberVar = 30
//
// === 调用方法 ===
// 局部变量 localVar = 0
// 块局部变量 blockLocalVar = 50
//
// staticVar = 100
// obj1.memberVar = 30
// obj2.memberVar = 30

// 初始值对比
public class DefaultValues {
    static int staticInt;           // 0
    static boolean staticBoolean;   // false
    static String staticString;     // null

    int memberInt;       // 0
    boolean memberBoolean;  // false
    String memberString;    // null

    public void method() {
        int localInt;           // 错误：未初始化，必须赋值才能使用
        // System.out.println(localInt);  // 编译错误

        int localInt2 = 0;      // 正确
    }
}

// 执行顺序演示
public class ExecutionOrder {
    static {
        System.out.println("1. 静态初始化块");
    }

    {
        System.out.println("2. 非静态初始化块");
    }

    public ExecutionOrder() {
        System.out.println("3. 构造方法");
    }

    public static void main(String[] args) {
        System.out.println("开始");
        new ExecutionOrder();
        new ExecutionOrder();
    }
}

// 输出：
// 1. 静态初始化块
// 开始
// 2. 非静态初始化块
// 3. 构造方法
// 2. 非静态初始化块
// 3. 构造方法
```

---

### 14. Java 中 String.length() 的运作原理

#### 详细答案

`String.length()` 返回的是字符串中的**字符数量**，而不是字节数。对于 ASCII 字符，1 个字符 = 1 字节；对于中文等，1 个字符 = 2-4 字节。

#### 代码示例

```java
public class StringLengthDemo {

    public static void main(String[] args) throws UnsupportedEncodingException {
        // 1. 基本用法
        String str = "Hello";
        System.out.println(str.length());  // 5

        // 2. 中文字符
        String chinese = "你好";
        System.out.println(chinese.length());  // 2（是字符数，不是字节数）

        // 3. 获取字节数
        System.out.println("UTF-8 字节数: " + chinese.getBytes("UTF-8").length);  // 6 (每个中文字符3字节)
        System.out.println("GBK 字节数: " + chinese.getBytes("GBK").length);    // 4 (每个中文字符2字节)
        System.out.println("Unicode 字节数: " + chinese.getBytes("Unicode").length);  // 6

        // 4. 特殊字符（Emoji）
        String emoji = "😀";
        System.out.println(emoji.length());  // 2（占用两个 char 单位）
        System.out.println(emoji.codePointCount(0, emoji.length()));  // 1（实际的 Unicode 字符数）

        // 5. 空字符串
        String empty = "";
        System.out.println(empty.length());  // 0

        String nullStr = null;
        // System.out.println(nullStr.length());  // NullPointerException

        // 6. String.length() 源代码实现
        // public int length() {
        //     return value.length;
        // }
        // 其中 value 是 private final char[] value;
        // 所以 length() 返回的就是内部 char 数组的长度
    }
}

// 字符编码详解
public class CharacterEncoding {

    public static void main(String[] args) throws Exception {
        String str = "Hello你好😀";

        System.out.println("字符数（char 数）: " + str.length());  // 8

        System.out.println("Unicode 字符数: " + str.codePointCount(0, str.length()));  // 7

        byte[] utf8 = str.getBytes("UTF-8");
        System.out.println("UTF-8 字节数: " + utf8.length);  // 5 + 6 + 4 = 15

        byte[] gbk = str.getBytes("GBK");
        System.out.println("GBK 字节数: " + gbk.length);  // 5 + 4 + 2 = 11（GBK 不支持 Emoji）

        // 字节到字符的转换
        String reconstructed = new String(utf8, "UTF-8");
        System.out.println(reconstructed);  // Hello你好😀
    }
}

// 性能提示
public class StringLengthPerformance {

    public static void main(String[] args) {
        String str = "Hello World";

        // 1. String.length() 是 O(1) 操作
        long start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            str.length();  // 直接返回数组长度，很快
        }
        long end = System.nanoTime();
        System.out.println("String.length() 耗时: " + (end - start) / 1000 + " µs");

        // 2. 获取字节数需要转换，较慢
        start = System.nanoTime();
        for (int i = 0; i < 1000000; i++) {
            try {
                str.getBytes("UTF-8").length;  // 需要编码转换
            } catch (Exception e) {}
        }
        end = System.nanoTime();
        System.out.println("getBytes().length 耗时: " + (end - start) / 1000 + " µs");
    }
}

// 常见错误
public class StringLengthMistakes {

    public static void main(String[] args) throws Exception {
        // 错误1：混淆字符数和字节数
        String name = "王";
        System.out.println(name.length());  // 1（字符数）
        System.out.println(name.getBytes("UTF-8").length);  // 3（字节数）

        // 错误2：不考虑 Emoji 占用多个 char
        String emoji = "👨‍👩‍👧‍👦";  // Family: man, woman, girl, boy
        System.out.println(emoji.length());  // 不是 1，会更大
        System.out.println(emoji.codePointCount(0, emoji.length()));  // 实际字符数

        // 错误3：遍历时没有考虑高代理对
        for (int i = 0; i < emoji.length(); i++) {
            char c = emoji.charAt(i);  // 某些字符可能是错误的
        }

        // 正确的遍历方式
        int[] codePoints = emoji.codePoints().toArray();
        for (int codePoint : codePoints) {
            System.out.println(Character.toString(codePoint));
        }
    }
}
```

---

## 📌 Java 集合部分

### 1. List、Set 以及 Map 的区别

#### 详细答案

| 特性 | List | Set | Map |
|------|------|-----|-----|
| 接口 | Collection 子接口 | Collection 子接口 | 独立接口 |
| 顺序 | **有序** | **无序** | **无序（LinkedHashMap 有序）** |
| 重复 | **允许重复** | **不允许重复** | key 不重复，value 可重复 |
| 实现类 | ArrayList、LinkedList | HashSet、LinkedHashSet | HashMap、Hashtable |
| 线程安全 | List 默认不安全 | Set 默认不安全 | Map 默认不安全 |
| 常用方法 | add、get、remove | add、contains、remove | put、get、keySet |

```
Collection
├── List（有序，允许重复）
│   ├── ArrayList
│   ├── LinkedList
│   └── Vector
├── Set（无序，不重复）
│   ├── HashSet
│   ├── LinkedHashSet
│   └── TreeSet
└── Queue
    ├── PriorityQueue
    └── Deque

Map（Key-Value）
├── HashMap
├── LinkedHashMap
├── TreeMap
├── Hashtable
└── ConcurrentHashMap
```

#### 代码示例

```java
public class CollectionDemo {

    public static void main(String[] args) {
        // 1. List 示例
        List<String> list = new ArrayList<>();
        list.add("Apple");
        list.add("Banana");
        list.add("Apple");  // 允许重复

        System.out.println("List: " + list);  // [Apple, Banana, Apple]
        System.out.println("List 大小: " + list.size());  // 3
        System.out.println("List 包含 Banana: " + list.contains("Banana"));  // true

        // 2. Set 示例
        Set<String> set = new HashSet<>();
        set.add("Apple");
        set.add("Banana");
        set.add("Apple");  // 重复元素会被忽略

        System.out.println("Set: " + set);  // [Apple, Banana] 或 [Banana, Apple]
        System.out.println("Set 大小: " + set.size());  // 2
        System.out.println("Set 包含 Banana: " + set.contains("Banana"));  // true

        // 3. Map 示例
        Map<String, Integer> map = new HashMap<>();
        map.put("Alice", 25);
        map.put("Bob", 30);
        map.put("Alice", 26);  // 覆盖原值

        System.out.println("Map: " + map);  // {Alice=26, Bob=30}
        System.out.println("Map 大小: " + map.size());  // 2
        System.out.println("Alice 的年龄: " + map.get("Alice"));  // 26

        // 4. 有序和无序的演示
        System.out.println("\n=== 有序性演示 ===");

        List<String> orderedList = new ArrayList<>();
        orderedList.add("1");
        orderedList.add("2");
        orderedList.add("3");
        System.out.println("List（有序）: " + orderedList);  // [1, 2, 3]

        Set<String> unorderedSet = new HashSet<>();
        unorderedSet.add("1");
        unorderedSet.add("2");
        unorderedSet.add("3");
        System.out.println("HashSet（无序）: " + unorderedSet);  // 可能 [1, 3, 2] 或其他顺序

        Set<String> orderedSet = new LinkedHashSet<>();
        orderedSet.add("1");
        orderedSet.add("2");
        orderedSet.add("3");
        System.out.println("LinkedHashSet（有序）: " + orderedSet);  // [1, 2, 3]

        // 5. 线程安全的集合
        List<String> synchronizedList = Collections.synchronizedList(new ArrayList<>());
        Set<String> synchronizedSet = Collections.synchronizedSet(new HashSet<>());
        Map<String, String> synchronizedMap = Collections.synchronizedMap(new HashMap<>());

        // 或者使用 Concurrent 集合
        Map<String, String> concurrentMap = new ConcurrentHashMap<>();
    }
}

// 选择 Collection 的建议
public class CollectionChoosing {

    public static void main(String[] args) {
        // 需要顺序且频繁随机访问 → ArrayList
        List<String> list1 = new ArrayList<>();
        list1.add("A");
        System.out.println(list1.get(0));  // 快

        // 需要顺序且频繁插入删除 → LinkedList
        LinkedList<String> list2 = new LinkedList<>();
        list2.add("A");
        list2.remove(0);  // 在 LinkedList 中效率高

        // 需要不重复的无序集合 → HashSet
        Set<String> set1 = new HashSet<>();
        set1.add("A");
        set1.add("B");
        boolean exists = set1.contains("A");  // 快

        // 需要不重复且有序的集合 → LinkedHashSet
        Set<String> set2 = new LinkedHashSet<>();

        // 需要不重复且排序的集合 → TreeSet
        Set<String> set3 = new TreeSet<>();

        // 需要 Key-Value 映射且快速查询 → HashMap
        Map<String, Integer> map1 = new HashMap<>();
        map1.put("Alice", 25);
        System.out.println(map1.get("Alice"));  // 快

        // 需要 Key 有序的映射 → LinkedHashMap 或 TreeMap
        Map<String, Integer> map2 = new LinkedHashMap<>();  // 插入顺序
        Map<String, Integer> map3 = new TreeMap<>();  // 自然顺序
    }
}
```

---

### 2. ArrayList 和 LinkedList 的区别

#### 详细答案

| 操作 | ArrayList | LinkedList |
|------|-----------|-----------|
| 随机访问 | **O(1)** 快 | O(n) 慢 |
| 在末尾插入 | O(1) | **O(1)** |
| 在中间插入 | O(n) | **O(1)** |
| 删除 | O(n) | O(n) |
| 内存占用 | 较少 | 较多（每个元素需要前后指针） |
| 实现 | 动态数组 | 双链表 |

#### 代码示例

```java
public class ArrayListVsLinkedList {

    public static void main(String[] args) {
        // 1. 访问性能对比
        System.out.println("=== 访问性能对比 ===");
        int size = 1000000;

        ArrayList<Integer> arrayList = new ArrayList<>();
        LinkedList<Integer> linkedList = new LinkedList<>();

        for (int i = 0; i < size; i++) {
            arrayList.add(i);
            linkedList.add(i);
        }

        // ArrayList 随机访问快
        long start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            arrayList.get(i);
        }
        long end = System.nanoTime();
        System.out.println("ArrayList get: " + (end - start) / 1000000 + "ms");

        // LinkedList 随机访问慢
        start = System.nanoTime();
        for (int i = 0; i < size; i++) {
            linkedList.get(i);
        }
        end = System.nanoTime();
        System.out.println("LinkedList get: " + (end - start) / 1000000 + "ms");

        // 2. 插入性能对比
        System.out.println("\n=== 插入性能对比 ===");

        // ArrayList 在末尾插入快
        arrayList = new ArrayList<>();
        start = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            arrayList.add(i);
        }
        end = System.nanoTime();
        System.out.println("ArrayList add: " + (end - start) / 1000000 + "ms");

        // LinkedList 在末尾插入也快
        linkedList = new LinkedList<>();
        start = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            linkedList.add(i);
        }
        end = System.nanoTime();
        System.out.println("LinkedList add: " + (end - start) / 1000000 + "ms");

        // ArrayList 在中间插入慢
        arrayList = new ArrayList<>();
        for (int i = 0; i < 10000; i++) {
            arrayList.add(i);
        }
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            arrayList.add(5000, i);  // 在中间插入
        }
        end = System.nanoTime();
        System.out.println("ArrayList insert middle: " + (end - start) / 1000000 + "ms");

        // LinkedList 在中间插入快
        linkedList = new LinkedList<>();
        for (int i = 0; i < 10000; i++) {
            linkedList.add(i);
        }
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) {
            linkedList.add(5000, i);  // 在中间插入
        }
        end = System.nanoTime();
        System.out.println("LinkedList insert middle: " + (end - start) / 1000000 + "ms");
    }
}

// ArrayList 内部实现原理
public class ArrayListInternal {

    public static class SimpleArrayList<E> {
        private Object[] elementData = new Object[10];
        private int size = 0;

        // 添加元素
        public void add(E e) {
            // 扩容检查
            if (size == elementData.length) {
                expandCapacity();
            }
            elementData[size++] = e;
        }

        // 扩容（1.5倍）
        private void expandCapacity() {
            Object[] newArray = new Object[elementData.length + (elementData.length >> 1)];
            System.arraycopy(elementData, 0, newArray, 0, elementData.length);
            elementData = newArray;
        }

        // 随机访问
        @SuppressWarnings("unchecked")
        public E get(int index) {
            return (E) elementData[index];
        }

        // 在中间插入（需要移动后面的元素）
        public void add(int index, E e) {
            if (size == elementData.length) {
                expandCapacity();
            }
            // 后面的元素向后移动
            System.arraycopy(elementData, index, elementData, index + 1, size - index);
            elementData[index] = e;
            size++;
        }
    }
}

// LinkedList 内部实现原理
public class LinkedListInternal {

    public static class SimpleLinkedList<E> {
        private Node<E> head;
        private int size = 0;

        private static class Node<E> {
            E data;
            Node<E> prev;
            Node<E> next;

            Node(E data) {
                this.data = data;
            }
        }

        // 在末尾添加
        public void add(E e) {
            if (head == null) {
                head = new Node<>(e);
            } else {
                Node<E> current = head;
                while (current.next != null) {
                    current = current.next;
                }
                Node<E> newNode = new Node<>(e);
                current.next = newNode;
                newNode.prev = current;
            }
            size++;
        }

        // 在指定位置插入
        public void add(int index, E e) {
            Node<E> node = getNode(index);
            Node<E> newNode = new Node<>(e);

            if (node.prev != null) {
                node.prev.next = newNode;
            }
            newNode.prev = node.prev;
            newNode.next = node;
            node.prev = newNode;

            if (node == head) {
                head = newNode;
            }
            size++;
        }

        // 获取指定位置的节点（需要遍历）
        private Node<E> getNode(int index) {
            Node<E> current = head;
            for (int i = 0; i < index; i++) {
                current = current.next;
            }
            return current;
        }

        // 访问元素（需要遍历）
        @SuppressWarnings("unchecked")
        public E get(int index) {
            return getNode(index).data;
        }
    }
}

// 使用建议
public class UsageGuide {

    public static void main(String[] args) {
        // 场景1：需要频繁随机访问 → ArrayList
        List<String> names = new ArrayList<>();
        names.add("Alice");
        names.add("Bob");
        System.out.println(names.get(0));  // 推荐用 ArrayList

        // 场景2：需要频繁在中间插入删除 → LinkedList
        List<String> queue = new LinkedList<>();
        queue.add(0, "first");  // 推荐用 LinkedList
        queue.remove(0);

        // 场景3：频繁遍历 → ArrayList
        for (String name : names) {
            System.out.println(name);
        }

        // 场景4：作为栈使用 → LinkedList
        LinkedList<Integer> stack = new LinkedList<>();
        stack.push(1);
        stack.push(2);
        int top = stack.pop();
    }
}
```

---

（继续生成其他集合题目...）

---

## 📌 Java 多线程部分

（由于篇幅限制，多线程部分内容较多，会在下一个文件中继续）

---

## 📌 说明

这是 Java 知识的完整学习笔记，包含：
- ✅ 详细的理论解释
- ✅ 表格对比
- ✅ 代码示例
- ✅ 应用场景说明
- ✅ 常见错误提示

建议学习方法：
1. **理论阶段**：阅读详细答案 + 理解代码示例
2. **实践阶段**：手写代码，运行示例
3. **复习阶段**：结合快速复习卡快速回顾

---
