---
theme: z-blue
---

在 TypeScript 中，`class`、`interface` 和 `type` 这三者经常让人混淆，尤其是当它们与 `extends`（继承）和 `implements`（实现）这两个关键字搭配使用时，本文将对这五个概念进行梳理。

## 一、 定义类型的方式

### 1. `class` (类)
- **双重身份**：它**既是值，也是类型**。
- **作为值**：它是一个构造函数，在代码运行时（Runtime）真实存在。
- **作为类型**：它是实例的蓝图，描述了对象的形状（Shape）。

```typescript
class Person {
  name: string;
  age: number;
}

const p: Person = new Person(); // 第一个 Person 是类型，第二个 Person 是值
```

### 2. `interface` (接口)
- **身份**：**纯类型**。
- **作用**：它是“契约”，只描述对象的形状（Shape），不包含具体实现。
- **本质**：编译后**完全消失**（变成空气），运行时不存在。
- **注意**：
    1. 接口里定义的所有属性和方法，**默认且强制是 `public` 的**，接口内不能描述类的**私有成员**（Private）和**受保护成员**（Protected）。
    2. **不能混用**：描述**构造函数**（`new`）/ **静态成员**（`static`）的接口，与描述**实例成员**的接口必须分开定义，不能合并在同一个接口中。

```typescript
// 1. 描述实例形状（属性 + 函数）
interface Animal {
  name: string;    // 属性
  eat(): void;     // 函数
}

// 2. 描述构造函数（特殊用法：构造签名）
// 用于描述“类”本身，而不是实例
interface AnimalConstructor {
  new (name: string): Animal; // 构造签名
  genus: string;              // 静态属性 (static)
}
```

### 3. `type` (类型别名)
- **身份**：**纯类型**。
- **作用**：它是“万能定义器”，可以给对象、联合类型、基本类型、元组等起个名字。
- **本质**：编译后**完全消失**，运行时不存在。

```typescript
// 1. 基本用法
type Name = string;

// 2. 联合类型
type ID = string | number;

// 3. 交叉类型
// 注意：如果属性冲突（类型不兼容），会变成 never 类型
// 例如：type A = { x: string; }; type B = { x: number; }; type C = A & B; // x: never
type Manager = { name: string; } & { id: number; }; // { name: string; id: number; }

// 4. 对象类型 (类似 interface)
type User = {
  name: string;
  age: number;
};
```

---

## 二、 三者之间的核心区别

### 1. Class 与 Interface 的区别

-   **作为值**：
    -   `Class`：**是**构造函数。编译后保留，运行时存在。
    -   `Interface`：**不是**值。编译后完全消失，运行时不存在。
-   **作为类型**：
    -   两者在绝大部分情况下（即描述公开形状时）**类型等价**。但在以下两种情况**不等价**：
        1.  **私有/受保护成员**：`Class` 类型包含 `private`/`protected` 成员，而 `Interface` 只能描述 `public` 成员。
        2.  **构造函数**：`Class` 作为类型时会自动推导构造函数签名，而 `Interface` 必须手动显式定义 `new` 签名。

### 2. Interface 与 Type 的区别

它们在 95% 的描述对象场景下是通用的，但在以下两个核心点上截然不同：

**A. 声明合并 (Interface 的独门绝技)**
- **Interface 是开放的**：同名 interface 会自动合并。这在给第三方库（如 `Window`）补丁属性时非常有用。
- **Type 是封闭的**：同名 type 会报错（重复定义）。

```typescript
// Interface 自动合并
interface User { name: string; }
interface User { age: number; }
// 最终 User 既有 name 也有 age

// Type 报错
type Dog = { name: string };
type Dog = { age: number }; // ❌ 报错
```

**B. 表达能力 (Type 的管辖范围更宽)**
`interface` 只能描述对象或函数。一旦涉及到 **联合类型 (|)**、**基本类型别名**、**元组**，必须用 `type`。

```typescript
type ID = string | number;       // Interface 做不到
type Status = 'open' | 'close';  // Interface 做不到
```

---

## 三、 建立关系的手段

### 1. `extends` (继承 / 扩展)
- **目的**：**复用**（Class）或**合并**（Interface）现有的属性或方法。
- **细分场景 (9种组合全解析)**：

    **A. Class 作为发起者 (`class A extends B`)**

    1.  **Class extends Class**：✅ **核心用法**。
        -   传统的面向对象语法，用于复用代码逻辑。
    2.  **Class extends Interface**：❌ **不支持**。
        -   类不能继承接口，只能**实现**接口（请用 `implements`）。
    3.  **Class extends Type**：❌ **不支持**。
        -   同上（请用 `implements`）。

    **B. Interface 作为发起者 (`interface A extends B`)**

    4.  **Interface extends Class**：⚠️ **合法但少见**。
        -   不建议使用。
    5.  **Interface extends Interface**：✅ **核心用法**。
        -   类型合并。支持多重继承（`extends A, B`）。
    6.  **Interface extends Type**：✅ **支持**。
        -   类型合并。仅当 Type 描述的是对象结构时支持。

    **C. Type 作为发起者 (`type A extends B`)**

    7.  **Type extends Class**：❌ **无此语法**。
    8.  **Type extends Interface**：❌ **无此语法**。
    9.  **Type extends Type**：❌ **无此语法**。

### 2. `implements` (实现)
- **目的**：**约束**，强制检查某个类是否包含了指定的属性或方法。
- **细分场景 (9种组合全解析)**：

    **A. Class 作为发起者 (`class A implements B`)**

    1.  **Class implements Class**：⚠️ **合法但少见**。
        -   不建议使用。
    2.  **Class implements Interface**：✅ **核心用法**。
        -   最常见的契约约束方式。
    3.  **Class implements Type**：✅ **支持**。
        -   仅当该 Type 描述的是对象结构。

    **B. Interface 作为发起者 (`interface A implements B`)**

    4.  **Interface implements Class**：❌ **不支持**。
    5.  **Interface implements Interface**：❌ **不支持**。
    6.  **Interface implements Type**：❌ **不支持**。
        -   接口只能声明结构，不能实现逻辑，所以没有 `implements` 关键字（请用 `extends`）。

    **C. Type 作为发起者 (`type A implements B`)**

    7.  **Type implements Class**：❌ **无此语法**。
    8.  **Type implements Interface**：❌ **无此语法**。
    9.  **Type implements Type**：❌ **无此语法**。

---

## 四、 核心关系矩阵（排列组合）

这是最容易混淆的部分，请参考下表：

| 发起者 (左) | 动作 | 目标 (右) | 解释 | 常见度 |
| :--- | :--- | :--- | :--- | :--- |
| **Class** | **extends** | **Class** | **继承**。子类继承父类的具体代码实现（原型链串联）。 | ⭐⭐⭐⭐⭐ |
| **Interface** | **extends** | **Interface** | **扩展**。接口继承另一个接口的属性（合并）。 | ⭐⭐⭐⭐⭐ |
| **Interface** | **extends** | **Type** | **扩展**。接口继承 Type 的属性（合并，仅当 Type 为对象结构时）。 | ⭐⭐⭐⭐ |
| **Class** | **implements** | **Interface** | **实现**。类必须满足接口定义的形状契约。 | ⭐⭐⭐⭐⭐ |
| **Class** | **implements** | **Type** | **实现**。类必须满足 Type 定义的形状契约（仅当 Type 为对象结构时）。 | ⭐⭐⭐⭐ |

**总结一句话：**
- **类继承类**：传统面向对象，复用代码逻辑
- **接口继承**：类型合并，扩展结构
- **类实现接口**：类型约束，满足契约