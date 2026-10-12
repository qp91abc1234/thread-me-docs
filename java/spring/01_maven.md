## 定义

Java 项目常用的项目构建和依赖管理工具

```
Maven
│
├── ① 依赖管理
│      ↓
│   管理项目需要的第三方 jar 包
│
└── ② 项目构建
       ↓
    编译 → 测试 → 打包 → 安装 / 发布
```

## pom.xml

核心配置文件

```
pom.xml
│
├── 项目信息
│   └── 我是谁、版本多少
│
├── 依赖管理
│   └── 我需要哪些 jar
│
└── 构建配置
    └── 怎么编译、测试、打包……
```

### GAV

是 Maven 用来唯一标识一个项目/依赖的核心坐标

```
G → groupId：组织/项目组，一般类似 Java 包名，例如 com.baomidou
A → artifactId：具体模块/产物名称，例如 mybatis-plus-spring-boot3-starter
V → version：版本，例如 3.5.12

<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.12</version>
</dependency>
```

## 常用命令

| 命令 | 作用 |
|---|---|
| `mvn clean` | 删除 `target` 构建目录，清理旧构建产物 |
| `mvn compile` | 编译项目主代码 |
| `mvn test` | 编译并运行测试代码 |
| `mvn package` | 编译、测试并打包，通常生成 `.jar` |
| `mvn clean package` | **最常用之一**，清理后重新完整打包 |
| `mvn install` | 构建后把当前项目产物安装到 Maven 本地仓库 |
| `mvn clean install` | 清理、构建并安装到本地仓库，多模块项目常见 |
| `mvn spring-boot:run` | 直接通过 Maven 启动 Spring Boot 项目 |
| `mvn dependency:tree` | **很实用**，查看完整依赖树，排查依赖冲突 |