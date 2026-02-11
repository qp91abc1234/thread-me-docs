---
theme: z-blue
---

## Sass 模块内容

一个 Sass 模块被加载时，会涉及两类内容：**成员** 和 **顶层 CSS**。

### 成员

- **是什么**：变量（`$xxx`）、mixin（`@mixin xxx`）、函数（`@function xxx`）
- **特点**：属于模块的「公开 API」，通过**命名空间**访问（如 `rb.responsive-bg`）
- **何时参与编译**：只有在你**调用 / 使用**它们时才会参与编译（例如 `@include rb.responsive-bg(...)`）

### 顶层 CSS

- **是什么**：直接写在文件顶层、不在 mixin/function 里的规则，例如 `.title { font-size: 20px; }`
- **特点**：不能通过命名空间访问（不能写 `rb.title`）
- **何时参与编译**：所在文件**被加载（执行）一次**时，这些 CSS 就会一起输出到编译结果里

---

## @use 做了什么

当 **A 文件** 写 `@use 'B' as b` 时，会发生两件事：

1. **执行 B 文件**  
   B 被加载并执行一次 → B 里的**顶层 CSS** 会出现在最终生成的 CSS 里（和 A 自己编译出的样式一起）。

2. **把 B 的成员暴露给 A**  
   A 里可以通过命名空间使用 B 的变量、mixin、函数，例如：`b.$color`、`@include b.box`、`b.some-function(...)`。

所以：**A use 了 B → A 既「带上」了 B 的顶层 CSS，又可以通过模块名使用 B 的成员。**

**b.scss：**

```scss
$color: red;

@mixin box {
  padding: 10px;
}

.title { font-size: 20px; }
```

**a.scss：**

```scss
@use 'b' as b;

.page {
  @include b.box;
  color: b.$color;
}
```

**A 的编译结果中会包含：**

- B 的顶层 CSS：`.title { font-size: 20px; }`
- A 使用 B 的成员后产生的：`.page { padding: 10px; color: red; }`

---

## @forward 做了什么

当 **B 文件** 写 `@forward 'C'` 时，会发生两件事：

1. **执行 C 文件**  
   C 作为 B 的依赖被加载并执行一次 → C 里的**顶层 CSS** 会出现在编译结果里（和 B 自己编译出的样式一起）。

2. **把 C 的成员放到 B 的命名空间下**  
   B 不能自己访问 C 的成员，只负责转发；谁 `@use 'B' as b`，谁才可以通过 B 的命名空间访问 B 和 C 的成员，例如：`b.$color`、`@include b.box`。

所以：**B forward 了 C → 谁 use B，谁就既「带上」了 B 和 C 的顶层 CSS，又可以通过 B 的命名空间使用 B 和 C 的成员。**

**和 @use 的区别**：`@use` 是「当前文件自己要**用**那个模块」；`@forward` 是「当前文件**自己不用**，而是让**别人在 use 当前文件时，也能用到**被 forward 的模块」。

**c.scss：**

```scss
$color: red;

@mixin box {
  padding: 10px;
}

.title { font-size: 20px; }
```

**b.scss（入口，只做转发）：**

```scss
@forward 'c';
```

**a.scss：**

```scss
@use 'b' as b;

.page {
  @include b.box;
  color: b.$color;
}
```

**B 的编译结果中会包含：**

- C 的顶层 CSS：`.title { font-size: 20px; }`

**A 的编译结果中会包含：**

- C 的顶层 CSS：`.title { font-size: 20px; }`
- A 使用 B 的成员后产生的：`.page { padding: 10px; color: red; }`（`b.box`、`b.$color` 实际来自 C）

---

## 小结

| 类型 | 怎么进编译结果 | @use | @forward |
|------|----------------|------|----------|
| 变量/mixin/function | 只有被**调用**时才参与编译 | 当前文件用 `命名空间.成员名` 使用 | 再暴露给「use 当前文件」的人用 |
| 顶层 CSS（如 `.xx { }`） | 所在文件**被加载一次**就会输出 | 被 use 的文件会执行，其顶层 CSS 会输出 | 被 forward 的文件会作为依赖加载一次，其顶层 CSS 也会输出 |
