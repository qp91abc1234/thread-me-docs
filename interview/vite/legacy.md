## 兼容旧版本浏览器

### 兼容策略（Vite）
Vite 采用**双重构建策略**，兼顾现代浏览器的性能和旧版浏览器的兼容性。

#### 1. 默认构建（Modern Bundle）
- **目标**：支持原生 ESM 的现代浏览器（默认 `build.target: 'modules'`）。
- **原理**：利用 **esbuild** 进行高速转译，将 TS/JSX 等转为原生 ESM 代码。
- **产物**：体积小、运行快，通过 `<script type="module">` 加载。

#### 2. 传统构建（Legacy Bundle）
通过 `@vitejs/plugin-legacy` 插件实现对旧版浏览器（如 IE11）的支持。
- **目标**：不支持 ESM 或某些新特性的旧版浏览器（由 `browserslist` 决定）。
- **产物**：包含 Polyfill 的 SystemJS 包，通过 `<script nomodule>` 加载。
- **依赖**：必须安装 **Terser**，因为 esbuild 不支持压缩 ES5 代码。

### 核心技术原理

#### JavaScript 兼容（Babel）
Legacy 插件底层使用 Babel 进行降级处理：
- **功能**：
  - **语法转换（Transpilation）**：将高阶语法（如箭头函数、class）降级为 ES5。
  - **垫片填充（Polyfill）**：自动检测代码使用的 API（如 Promise、Map），按需引入 `core-js` 实现。
- **工作流程**：
  1. **Parse**：解析器将源码转为抽象语法树（AST）。
  2. **Transform**：遍历 AST，调用插件进行增删改（如 `babel-preset-env`）。
  3. **Generate**：将 AST 生成目标代码和 SourceMap。
- **注意**：显式添加 `regenerator-runtime` 可作为 async/await 支持的兜底保险。

#### 样式兼容（PostCSS）
CSS 的兼容性处理主要依赖 PostCSS 生态：
- **postcss-preset-env**：将现代 CSS 特性转换为大多数浏览器支持的 CSS（如 8位 hex 颜色转换）。
- **autoprefixer**：根据 `browserslist` 自动添加浏览器前缀（如 `-webkit-`, `-ms-`）。

#### 目标环境配置（Browserslist）
通过 `.browserslistrc` 文件统一管理兼容目标：
- **作用**：同时控制 Babel（JS 降级）和 PostCSS（CSS 前缀）的兼容范围。
- **示例**：`ie >= 11, chrome >= 52`。