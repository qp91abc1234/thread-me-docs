```java
遇到字符串操作
    │
    ├─ 是简单拼接（少于5个变量，非循环）？
    │   └─ 是 → 直接用 String 的 "+" 或 concat()
    │          (编译器自动优化，清晰优先)
    │
    ├─ 是在循环内拼接 / 构建复杂SQL/JSON？
    │   └─ 是 → 手动使用 StringBuilder
    │          (性能优先，注意预分配容量)
    │
    └─ 是需要判空、集合转字符串、路径处理等"辅助功能"？
        └─ 是 → 查看 Spring StringUtils 是否有现成方法
               (有就用，没有就自己写几行简单逻辑)
```

## String

不可变

```java
String str = " Hello, World! ";

// 判空（注意 null 判断）
str == null;                    // 判断是否为空引用
str.isEmpty();                  // 判断长度是否为0：""
str.isBlank();                  // JDK11+，判断是否空白："  " → true

// 长度
str.length();                   // 返回字符长度

// 截取
str.substring(2);               // "Hello, World! "
str.substring(2, 7);            // "Hello" (从2到7，不包含7)

// 查找
str.indexOf("World");           // 7，返回第一次出现位置
str.lastIndexOf("o");           // 11，最后一次出现位置
str.contains("Hello");          // true
str.startsWith(" He");          // true
str.endsWith("! ");             // true

// 比较
str.equals(" Hello, World! ");  // true，比较内容
str.equalsIgnoreCase(" hello, world! "); // true，忽略大小写
str.compareTo("Hello");         // 字典序比较

// 转换
str.toLowerCase();              // 转小写
str.toUpperCase();              // 转大写
str.trim();                     // 去除首尾空格（不会去除中间）
str.strip();                    // JDK11+，去除首尾空白（支持Unicode）
str.replace("World", "Java");   // 替换所有
str.replaceAll("\\s+", "");     // 正则替换
str.split(",");                 // 分割成数组

// 拼接（静态方法）
String.join("-", "a", "b", "c"); // "a-b-c"
String.join(",", list);          // 集合拼接

// 格式化
String.format("姓名：%s，年龄：%d", "张三", 18);

// 需要同时判断 null 和空字符串
if (str == null || str.isEmpty()) { }

// JDK11+ 判断 null 或空白（推荐）
if (str == null || str.isBlank()) { }

// 安全调用（避免空指针）
if ("hello".equals(str)) { }  // 常量在前
if (Objects.equals(str, "hello")) { }
```

## StringBuilder

主要用于 `循环内拼接` 和 `构建复杂的 SQL 或动态条件`

```java
StringBuilder sb = new StringBuilder();

sb.append("hello");

sb.append(" ");

sb.append("world");

String result = sb.toString();
```

### 底层实现

> StringBuilder 内部数组 byte[] value，在原数组上操作（仅扩容时创建新数组）
> String 内部数组 final byte[] value，修改操作每次创建新数组 + 新对象

```java
public final class StringBuilder {
    byte[] value;          // 没有 final！可以指向不同数组
    int count;             // 已使用的长度

    // 追加操作
    public StringBuilder append(String str) {
        int len = str.length();
        // 确保容量足够，不够就扩容
        ensureCapacity(count + len);
        // 直接把字符拷到现有数组后面
        str.getChars(0, len, value, count);
        count += len;
        return this;  // 返回自身，支持链式调用
    }

    // 扩容：创建更大的新数组，复制旧数据
    private void ensureCapacity(int minCapacity) {
        if (minCapacity - value.length > 0) {
            // 新容量 = 旧容量 * 2 + 2
            int newCapacity = value.length * 2 + 2;
            value = Arrays.copyOf(value, newCapacity);
        }
    }
}
```

### 为什么适合循环内字符串拼接

因为 String 是不可变的，循环内使用 + 拼接会不断创建新对象，而 StringBuilder 是可变的，全程只用一个对象

```java
// 源代码
String s = "a" + "b" + "c";

// 编译后的等效代码
String s = new StringBuilder().append("a").append("b").append("c").toString();
```

```java
// 源代码
String s = "";
for (int i = 0; i < 2; i++) {
    s = s + i;
}

// 第1次循环 (i=0)
StringBuilder sb1 = new StringBuilder();
sb1.append("");    // 原来的 s
sb1.append(0);     // i
s = sb1.toString(); // 产生新String对象 "0"

// 第2次循环 (i=1)
StringBuilder sb2 = new StringBuilder();
sb2.append("0");   // 原来的 s
sb2.append(1);     // i
s = sb2.toString(); // 产生新String对象 "01"
```

## StringUtils

Spring 框架自带的工具类

```java
// 是否有文本
StringUtils.hasText(null);      // false
StringUtils.hasText("");        // false
StringUtils.hasText("   ");     // false（纯空格）
StringUtils.hasText(" Hello "); // true

// 是否有长度
StringUtils.hasLength(null);    // false
StringUtils.hasLength("");      // false
StringUtils.hasLength("   ");   // true（注意：纯空格算有长度）
```
