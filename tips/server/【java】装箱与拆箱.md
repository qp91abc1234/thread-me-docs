# Java 装箱与拆箱

## 一、什么是装箱与拆箱

在 Java 中：

> **装箱（Boxing）和拆箱（Unboxing）是基本类型与包装类之间的自动转换机制**

---

## 二、基本类型与包装类

| 基本类型 | 包装类 |
|----------|--------|
| int      | Integer |
| double   | Double |
| boolean  | Boolean |
| char     | Character |

---

## 三、装箱（Boxing）

### 1. 定义

> **基本类型 → 包装类**

### 2. 示例

    int a = 10;
    Integer b = a;  // 自动装箱

### 3. 本质

    Integer b = Integer.valueOf(a);

---

## 四、拆箱（Unboxing）

### 1. 定义

> **包装类 → 基本类型**

### 2. 示例

    Integer b = 10;
    int a = b;  // 自动拆箱

### 3. 本质

    int a = b.intValue();

---

## 五、常见问题（重点）

### 1. 空指针异常

    Integer b = null;
    int a = b;  // 报错：NullPointerException

原因：

> 拆箱时会调用 `b.intValue()`，但 b 是 null

---

### 2. Integer 比较问题

    Integer a = 100;
    Integer b = 100;
    System.out.println(a == b); // true

    Integer a = 200;
    Integer b = 200;
    System.out.println(a == b); // false

原因：

- -128 ~ 127 使用缓存（IntegerCache）
- 超出范围创建新对象

正确写法：

    a.equals(b);

---

### 3. 性能问题（了解）

    for (int i = 0; i < 1000000; i++) {
        Integer x = i; // 频繁装箱
    }

---

## 六、为什么需要包装类

Java 是面向对象语言，但基本类型不是对象，因此需要包装类来支持更多场景。

---

## 七、必须使用包装类的三种情况（重点）

### 1. 集合中使用

    List<Integer> list = new ArrayList<>();
    list.add(10); // 自动装箱

---

### 2. 泛型限制

    Map<String, Integer> map = new HashMap<>();

错误示例：

    Map<String, int> map;

---

### 3. 需要表示 null（无值）

错误：

    int a = null;

正确：

    Integer a = null;

常用于：

- 数据库字段
- 接口参数
- JSON 数据

---

## 八、总结

- 装箱：基本类型 → 包装类
- 拆箱：包装类 → 基本类型
- 本质：自动调用 valueOf() 和 xxxValue()
- 包装类的核心作用：
  - 支持集合
  - 支持泛型
  - 支持 null

---

## 九、一句话理解

> 装箱 = 给基本类型套一层对象壳  
> 拆箱 = 从对象中取出基本值