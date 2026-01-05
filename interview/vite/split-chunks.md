## 代码分割策略

代码分割的主要好处：
- **提升缓存命中率**：将第三方依赖按功能分组打包，当某个依赖更新时，只需重新加载对应的 chunk，其他未变更的 chunk 仍可使用缓存
- **减少加载无用代码**：通过按页面分包和动态导入，用户只需加载当前页面所需的代码，避免一次性加载所有页面代码，减少首屏加载时间

### 第三方依赖分包（manualChunks）
通过配置 `rollupOptions.output.manualChunks` 实现第三方依赖的代码分割
- **分包规则**：根据模块路径中的关键词匹配，将第三方依赖分组打包
- **分包策略**：
  - `vue-core`：Vue 核心库（vue、vue-router、pinia）
  - `ui-lib`：UI 组件库（element-plus、@element-plus/icons-vue）
  - `vueuse`：Vue 工具库（@vueuse）
  - `network-date`：网络请求和时间处理（axios、dayjs）
  - `utility`：工具库（lodash-es、crypto-js、uuid、nanoid、mitt）
  - `vconsole`：调试工具（vconsole）
  - `vendor`：其他未匹配的 node_modules 依赖
- **匹配逻辑**：通过检查模块路径是否包含 `/node_modules/{keyword}/` 或 `/node_modules/{keyword}` 来判断归属
- **优化策略**：
  - 将第三方依赖按功能分组，提升缓存命中率，减少重复打包
  - 对于只有某个页面依赖的大 npm 包，不放入 `vendor`，而是单独打包或通过动态导入实现按需加载

### 业务代码分包
- **公共代码分包（Shared Chunk）**：
  - Vite/Rollup 会自动识别多个 Chunk 共享的模块（如全局组件、工具函数）
  - 如果被多个页面引用且体积较大，会自动抽离成独立的 Shared Chunk，避免代码重复，实现跨页面缓存复用
  - 如果引用少或体积小，则直接内联到各 Chunk 中，减少 HTTP 请求
- **按页面分包**：通过异步路由（`component: () => import('@/views/xxx.vue')`）实现业务代码按页面分包，每个页面组件会被打包成独立的 chunk
- **页面内部二次拆分**：如果页面组件过大，可在组件内部使用动态模块导入（`defineAsyncComponent` 或 `import()`）进行二次拆分，将大组件单独打包
- **资源提示符（Resource Hints）优化**：对于单独打包的页面依赖模块，可在进入页面时动态添加 `<link rel="prefetch">` 标签到 HTML head 中，实现按需预取
  - 不在 HTML 中静态添加所有资源提示符，避免一次性预取所有资源，节省带宽
  - 可根据用户行为智能预测，预取可能访问的页面资源（如进入首页时预取常用页面）
  - 在浏览器空闲时预取，提升后续页面加载速度
