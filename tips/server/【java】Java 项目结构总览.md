# Java 项目结构总览（项目 / 模块 / 包 / 文件 / 类）

## 一、整体层级关系

> 项目 > 模块（Maven）> 包（package）> 文件（.java）> 类（class）

```
project（项目）
 ├── module（模块，Maven）
 │     ├── package（包）
 │     │     ├── Java 文件（.java）
 │     │     │     ├── class（类）
```

---

## 二、项目（Project）

> 一个完整的应用或系统

### 特点
- 包含多个模块
- 表示一个完整业务系统
- 由 Maven/Gradle 管理

---

## 三、模块（Module，Maven）

> 模块（module）是一个可独立构建、通过依赖关系与其他模块连接的工程单元，用于实现系统拆分和边界控制

### 特点
- 有独立 `pom.xml`
- 可单独编译、打包
- 模块之间必须通过依赖声明建立关系
- 是项目拆分和边界控制的核心单位

### 示例

```
project
 ├── user-module
 ├── order-module
 ├── common-module
```

---

## 四、包（package）

> 包 = 命名空间 + 代码组织 + 访问边界控制

### 1. 命名空间
- 避免类名冲突  
- 例如：`com.user.UserService` vs `com.order.UserService`

---

### 2. 代码组织
- 按业务或分层划分结构  
- 例如：
  - com.project.user
  - com.project.order

---

### 3. 访问边界控制
- 配合访问修饰符控制可见性  
- 同包更宽松，跨包更严格

---

### 4. 包与文件夹的关系（非常重要）

> ❗ 包名必须与文件夹结构一一对应  
> ❗ 包是逻辑组织方式，文件夹是其物理实现，二者不能分离

#### 示例

```java
package com.project.user.service;
```

对应目录结构：

```
src/main/java
 └── com/project/user/service/UserService.java
```

---

### 5. 包与 Java 文件的关系

> 一个包下可以有多个 Java 文件

#### 示例

```
com/project/user/
 ├── User.java
 ├── UserService.java
 ├── UserController.java
```

👉 这些文件都属于：

```java
package com.project.user;
```

---

## 五、类（class）

> 类 = 对象的模板（Blueprint）

### 定义内容
- 属性（变量）
- 行为（方法）

---

### Java 文件与类的关系

#### 规则

- 一个 `.java` 文件：
  - ✅ 只能有一个 `public class`
  - ✅ 文件名必须与该 `public class` 同名
- 一个文件可以有多个类（不推荐）：

```java
public class A {}

class B {}
class C {}
```

---

## 六、访问规则控制（类 + 成员）

### 1. 顶层类访问控制

| 修饰符 | 同包 | 跨包 |
|--------|------|------|
| public | ✅ | ✅ |
| 默认（不写） | ✅ | ❌ |

#### 关键点
- 非 public 类 = 仅包内可见
- `import` 不影响访问权限

---

### 2. 成员访问控制（变量 / 方法）

| 修饰符 | 同包 | 跨包 |
|--------|------|------|
| public | ✅ | ✅ |
| protected | ✅ | ⚠️ 子类 |
| 默认（不写） | ✅ | ❌ |
| private | ❌ | ❌ |

---

### 3. 访问规则总结

#### 同包
- 可以访问 **非 private 成员**

---

#### 跨包
必须同时满足：

- 类是 `public`
- 成员是 `public`

---

#### protected 特例

- 仅允许 **子类跨包访问**
- ❌ 不能通过父类对象访问
- ✅ 只能通过继承访问（`this` / `super`）

---

## 七、import 的本质

> import ≠ 获得访问权限

- 仅用于简化类名引用
- 是否能访问取决于：
  - 类修饰符
  - 成员修饰符

---

## 八、前端类比（用于快速理解）

| Java | 前端 |
|------|------|
| 项目（Project） | 仓库（repo） |
| 模块（module） | monorepo 子项目（workspace） |
| 包（package） | 文件夹 / ES module（类比） |
| 类（class） | JS/TS 文件 |
| Maven | npm + vite（+ 构建流程） |

> ❗ 类比仅用于理解，Java 更强调“强约束 + 编译期控制”

---

## 九、最终总结

> 项目是整体系统，模块（Maven）是拆分单位，包是代码组织结构，文件是类的载体，类是最小逻辑单元；  
> 模块通过依赖关系控制边界，包用于组织代码；  
> 包决定文件夹结构，文件决定 public 类，类决定程序行为；  
> 同包访问宽松，跨包访问严格，protected 仅对子类开放。
