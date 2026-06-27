## BIO

```java
BIO = 阻塞IO（Blocking IO）
         ↓
    读写时线程会"傻等"，直到数据读完才继续执行

核心类结构：
├── 字节流（处理一切，包括文本和二进制）
│   ├── InputStream（读）
│   │   ├── FileInputStream      ← 读文件
│   │   ├── BufferedInputStream  ← 带缓冲，性能更好（必用）
│   │   └── System.in            ← 控制台输入
│   └── OutputStream（写）
│       ├── FileOutputStream     ← 写文件
│       ├── BufferedOutputStream ← 带缓冲（必用）
│       └── System.out           ← 控制台输出
│
└── 字符流（只处理文本，避免乱码）
    ├── Reader（读）
    │   ├── FileReader           ← 读文本文件
    │   ├── BufferedReader       ← 带缓冲，按行读（必用）
    │   └── InputStreamReader    ← 字节→字符转换（指定编码）
    └── Writer（写）
        ├── FileWriter           ← 写文本文件
        ├── BufferedWriter       ← 带缓冲（必用）
        └── OutputStreamWriter   ← 字符→字节转换（指定编码）
```

### 读写文本文件

FileReader 不能指定编码！ 它永远使用"系统默认编码"（Windows是GBK，Linux是UTF-8）

通过转换流指定字符编码

```java
import java.io.*;
import java.nio.charset.StandardCharsets;

public List<String> readTextFile(String filePath) {
    List<String> lines = new ArrayList<>();
    // try-with-resources 自动关闭流（Java 7+）
    try (BufferedReader reader = new BufferedReader(
            new InputStreamReader(new FileInputStream(filePath), StandardCharsets.UTF_8))) {
        String line;
        while ((line = reader.readLine()) != null) {
            lines.add(line);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    return lines;
}

// 覆盖写入
public void writeTextFile(String filePath, String content) {
    try (BufferedWriter writer = new BufferedWriter(
            new OutputStreamWriter(new FileOutputStream(filePath), StandardCharsets.UTF_8))) {
        writer.write(content);
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// 追加写入（第二个参数 true）
public void appendTextFile(String filePath, String content) {
    try (BufferedWriter writer = new BufferedWriter(
            new OutputStreamWriter(new FileOutputStream(filePath, true), StandardCharsets.UTF_8))) {
        writer.newLine();  // 先换行
        writer.write(content);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### buffer 的作用

减少磁盘IO次数

```java
第1次 readLine():
    → 底层 FileReader 一次性读 8192 个字符到 char[] 缓存
    → 从缓存里找第一个 \n，返回一行
    → 指针移动到下一行开头

第2次 readLine():
    → 直接查缓存里还有没有数据
    → 有！从缓存里取，不访问硬盘
    → 返回第二行

...
缓存用完了（读到第 N 行时，缓存里的 8192 个字符用完了）：
    → 再次一次性读 8192 个字符到缓存
    → 继续返回行
```

## 文件操作

Files 类对文件内容的所有读写操作，底层全部依赖 Java 标准的 FileInputStream、FileOutputStream、FileChannel 等，而这些底层类在读取文件数据时，全都是阻塞的

```java
Path path = Paths.get("data.txt");

// 检查
Files.exists(path)                // 文件是否存在
Files.isDirectory(path)           // 是否是目录
Files.isRegularFile(path)         // 是否是普通文件
Files.isReadable(path)            // 是否可读
Files.isWritable(path)            // 是否可写

// 创建
Files.createFile(path);            // 创建空文件（文件已存在会抛异常）
Files.createDirectories(Paths.get("a/b/c")); // 创建目录（自动创建父目录）
Files.createTempFile("prefix", ".tmp"); // 创建临时文件

// 删除
Files.delete(path);                // 删除文件（不存在抛异常）
Files.deleteIfExists(path);        // 删除文件（不存在不抛异常）

Path source = Paths.get("source.txt");
Path target = Paths.get("backup/source.txt");

// 拷贝
Files.copy(source, target);                          // 简单拷贝
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING); // 覆盖已存在
Files.copy(source, System.out);                      // 拷贝到输出流（极简）

// 移动/重命名
Files.move(source, target, StandardCopyOption.REPLACE_EXISTING);

// 一次性读所有行（适合小文件 < 10MB）
List<String> lines = Files.readAllLines(Paths.get("data.txt"), StandardCharsets.UTF_8);

// 一次性写所有行
Files.write(Paths.get("output.txt"), lines, StandardCharsets.UTF_8);

// 追加内容
Files.write(Paths.get("log.txt"), "新日志".getBytes(StandardCharsets.UTF_8),
            StandardOpenOption.APPEND);

// 获取字节数组（图片等）
byte[] data = Files.readAllBytes(Paths.get("image.jpg"));

// 读文本文件（小文件，一次性）
public List<String> readSmallFile(String path) {
    try {
        return Files.readAllLines(Paths.get(path), StandardCharsets.UTF_8);
    } catch (IOException e) {
        log.error("读取文件失败", e);
        return Collections.emptyList();
    }
}

// 读文本文件（大文件，逐行）
public void readLargeFile(String path) {
    try (Stream<String> lines = Files.lines(Paths.get(path), StandardCharsets.UTF_8)) {
        lines.forEach(line -> {
            // 逐行处理
        });
    } catch (IOException e) {
        log.error("读取文件失败", e);
    }
}

// 写文本文件（小文件，大文件要用到 BufferedOutputStream）
public void writeFile(String path, String content) {
    try {
        Files.write(Paths.get(path), content.getBytes(StandardCharsets.UTF_8));
    } catch (IOException e) {
        log.error("写入文件失败", e);
    }
}

// 追加文本（小文件，大文件要用到 BufferedOutputStream）
public void appendFile(String path, String content) {
    try {
        Files.write(Paths.get(path), content.getBytes(StandardCharsets.UTF_8),
                    StandardOpenOption.CREATE, StandardOpenOption.APPEND);
    } catch (IOException e) {
        log.error("追加文件失败", e);
    }
}

// 拷贝文件
public void copyFile(String src, String dest) {
    try {
        Files.copy(Paths.get(src), Paths.get(dest), StandardCopyOption.REPLACE_EXISTING);
    } catch (IOException e) {
        log.error("拷贝文件失败", e);
    }
}

// 列出所有文件
try (Stream<Path> paths = Files.list(Paths.get("."))) {
    paths.filter(Files::isRegularFile)
         .forEach(System.out::println);
}

// 递归遍历所有.java文件
try (Stream<Path> paths = Files.walk(Paths.get("src"))) {
    paths.filter(p -> p.toString().endsWith(".java"))
         .forEach(System.out::println);
}

// 遍历最多2层深度
try (Stream<Path> paths = Files.walk(Paths.get("."), 2)) {
    // ...
}
```
