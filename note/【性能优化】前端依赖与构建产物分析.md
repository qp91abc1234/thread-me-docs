## 开发阶段（Dev）依赖链分析

### 目标
- 发现并处理二开过程中的无用引入、误引入、过早引入。

### 方法
- 使用 `vite-plugin-inspect` 观察模块转换与依赖关系（开发态视角）。
- 重点关注页面入口到业务模块的引入链，不追求全量模块图。

---

## 构建阶段（Build）产物分析

### 目标
- 分析不同场景下实际加载的构建产物（如首屏、路由切换、核心交互）。
- 定位体积大头和可优化点，指导性能优化。

### 工具
- `rollup-plugin-visualizer`：生成 treemap，用于查看各 chunk 的体积分布及其内部 module 构成。

### Visualizer 常用筛选
- **筛选 chunk + module**：`xxx-*.js:*/**/xxx.js`
- **仅筛选 chunk**：`*/xxx-*.js:`
- **仅筛选 module**：`*/**/xxx.js`

### 思路
1. 先拿到目标场景的真实请求集合（以浏览器网络日志为准）。
2. 再定位这些请求对应的 chunk。
3. 在 treemap 中分析 chunk 体积分布（谁大、占比多少）。
4. 回到源码判断优化策略：可移除、可替换、可拆分、可延后。
5. 完成优化后做前后对比（请求数、体积、关键路径时延）。

---

## `manifest`（产物依赖关系）

用于查看产物之间的 chunk 依赖关系，重点包括：
- 入口是谁（如 `index.html`）。
- 每个产物依赖哪些 chunk（`imports` / `dynamicImports`）。

### 启用方式

```ts
// vite.config.ts
{
  build: {
    manifest: true
  }
}
```

### 关于 `manifest` 中的 `index.html`
- `index.html` 在 `manifest` 中是入口标识（entry key）。
- 它对应的是入口 JS 产物（如 `assets/index-*.js`），不是运行时依赖 HTML 文件本体。
- 某动态页面的 `imports` 中出现 `index.html`，本质是该页面 chunk 依赖入口 chunk 的运行时代码或公共基础能力。
