# 构建过程中的对象：Bundle、Asset、Chunk

## 概念简述

Bundle、Asset、Chunk 是 **构建过程中的具体对象**（如 Rollup/Vite 构建 API 中的数据结构），与最终产出的文件一一对应：

- **Bundle**：构建过程中表示「整份构建产物」的集合对象，键为文件名，值为对应的 Asset 或 Chunk 对象。
- **Asset**：构建过程中表示「一份静态资源」的对象，与产物中的 HTML、CSS、图片、字体等非 JS 文件对应。
- **Chunk**：构建过程中表示「一份 JS 代码块」的对象，由若干 Module 编译、打包后生成，与产出的 `.js` 文件对应。

---

## Bundle

Bundle 是构建过程中的对象，表示一次构建的整份产出；以**文件名**为键、以 **Asset 或 Chunk** 为值的映射。写入磁盘后，就对应我们看到的 `dist/` 目录。

**对象结构示例：**

```js
{
  'index.html': { type: 'asset', fileName: 'index.html', source: '<!DOCTYPE html>...' },
  'assets/index-a1b2c3d4.js': { type: 'chunk', fileName: 'assets/index-a1b2c3d4.js', code: '...', ... },
  'assets/style-m3n4o5p6.css': { type: 'asset', fileName: 'assets/style-m3n4o5p6.css', source: '...' }
}
```

**对应的产物目录示例：**

```
dist/
├── index.html
├── assets/
│   ├── index-a1b2c3d4.js
│   ├── vendor-e5f6g7h8.js
│   ├── About-i9j0k1l2.js
│   ├── style-m3n4o5p6.css
│   └── logo-q7r8s9t0.png
```

---

## Asset

Asset 是构建过程中的对象，用来描述一份将要产出的静态资源；**不是**磁盘上的那个文件本身，而是与那份文件对应的数据结构。在 Rollup/Vite 里，每份静态资源输出都会有一个 Asset 对象。

**对象结构示例：**

```js
// 例如一个 CSS 文件对应的 Asset
{
  type: 'asset',
  fileName: 'assets/style-a1b2c3d4.css',
  source: 'body { margin: 0; }\n.app { color: red; }'  // 或 Uint8Array（二进制如图片）
}
```

**对应的产物常见类型：**

| 类型     | 示例                     |
|----------|--------------------------|
| HTML     | `index.html`             |
| CSS      | `assets/style-xxx.css`   |
| 图片     | `assets/logo.png`、`favicon.ico` |
| 字体     | `assets/font.woff2`      |
| 数据/配置 | `assets/data.json`、manifest 等 |

---

## Chunk

Chunk 是构建过程中的对象，用来描述一份将要产出的 JS 代码块；由一个或多个 Module 经依赖分析、合并、压缩后生成，与产出的 `.js` 文件对应。

**对象结构示例：**

```js
// 例如入口 Chunk
{
  type: 'chunk',
  fileName: 'assets/index-a1b2c3d4.js',
  name: 'index',
  code: 'import { x } from "./vendor-xxx.js";\n...',  // 产出的 JS 代码
  imports: ['assets/vendor-e5f6g7h8.js'],            // 静态依赖的 chunk 文件名
  dynamicImports: ['assets/About-i9j0k1l2.js'],      // 动态 import 的 chunk 文件名
  isEntry: true,
  isDynamicEntry: false
}
```

**对应的产物常见类型：**

| 类型       | 说明                               |
|------------|------------------------------------|
| 入口 Chunk | `index-xxx.js`，主入口打包结果     |
| 依赖 Chunk | `vendor-xxx.js`，如 node_modules   |
| 路由 Chunk | `About-xxx.js`，懒加载路由组件     |
| 异步 Chunk | 动态 `import()` 的组件对应 JS      |
