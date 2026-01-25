---
theme: z-blue
---

## 一、路径转换的定义

在 Vite 项目中，你在源码里写的资源路径（相对路径 `./logo.png`、别名路径 `@/assets/bg.jpg`）会在构建时被转换成浏览器可访问到的绝对路径（通常带 hash，如 `/assets/logo-a3f2c1.png`）。这个过程就是**路径转换**。

---

## 二、哪些路径会被转换

### ✅ 会转换的路径

#### 1. 相对路径（`./` 或 `../`）

```ts
// Script 中
import logoUrl from './assets/logo.png'

// Template 中（内置标签的静态资产属性）
<img src="./assets/logo.png" />

// CSS 中
background-image: url('./assets/bg.png');
```

#### 2. 别名路径（`@/` 等）

```ts
// 需要在 vite.config.ts 中配置
export default defineConfig({
  resolve: {
    alias: {
      '@': join(__dirname, 'src')
    }
  }
})

// Script 中
import logoUrl from '@/assets/logo.png'

// Template 中（内置标签的静态资产属性）
<img src="@/assets/logo.png" />

// CSS 中
background-image: url('@/assets/bg.png');
```

### ❌ 不会转换的路径

#### 1. 绝对路径（`/` 开头）

```ts
// Script 中
const url = '/logo.png'

// Template 中
<img src="/logo.png" />

// CSS 中
background-image: url('/bg.png');
```

**行为：**
- 不会被转换，直接请求 `public/` 目录
- 原样输出，不参与打包，不会加 hash

**适用场景：**
- 资源必须放在 `public/` 目录下
- 需要固定 URL（如 `favicon.ico`、`robots.txt`）

#### 2. 外部 URL 和 data URL

```html
<template>
  <!-- 外部 URL -->
  <img src="https://example.com/a.png" />
  
  <!-- data URL -->
  <img src="data:image/png;base64,iVBORw0K..." />
</template>
```

```scss
.bg {
  // 外部 URL
  background-image: url('https://example.com/bg.png');
  
  // data URL
  background-image: url('data:image/png;base64,iVBORw0K...');
}
```

**行为：** 不会转换，可以直接使用（它们本身就是完整的 URL 形态）。

#### 3. 变量/动态路径

```html
<script setup lang="ts">
const imagePath = './assets/logo.png'  // 变量
const name = 'logo'
</script>

<template>
  <!-- ❌ 不会转换，imagePath 是变量 -->
  <img :src="imagePath" />
  
  <!-- ❌ 不会转换，动态表达式 -->
  <img :src="`./assets/${name}.png`" />
</template>
```

**原因：** 路径转换是在编译时进行的，编译器无法确定变量的运行时值。

**解决方案：**
- 使用 `import.meta.glob()` 预先导入所有可能的文件
- 或将资源放入 `public/` 目录

---

## 三、new URL 的使用

`new URL(path, base)` 是 **ESM 标准 API**，用于 URL 解析（URL Resolution）。

**在 Vite 中的转换规则：**

### 相对路径
```ts
const url = new URL('./assets/icon.png', import.meta.url).href

// 开发时：'http://localhost:5173/src/assets/icon.png'
// 构建后：'/assets/icon-a3f2c1.png' ✅ 会转换并加 hash
```

### 别名路径
```ts
const url = new URL('@/assets/icon.png', import.meta.url).href

// ✅ 会转换（需要配置 resolve.alias）
// 构建后：'/assets/icon-a3f2c1.png'
```

### 绝对路径
```ts
const url = new URL('/logo.png', import.meta.url).href

// ❌ 不参与转换
// new URL 仅做 URL 解析（路径拼接），结果为同源绝对路径：'/logo.png'
```
