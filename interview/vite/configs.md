## Vite 常用配置 (Vite Config)

### 1. 基础配置 (Shared Options)
- **root**: Vite 以 HTML 文件为构建入口，会根据根 root 目录查找 `index.html` 并解析其中的依赖。默认为 `process.cwd()`。
- **base**: 公共基础路径（如 `/my-app/`），部署到非根目录时必须配置。
- **plugins**: 插件列表（如 `vue()`, `AutoImport()`）。
- **resolve.alias**: 路径别名（如 `@` -> `src`），简化导入路径。
- **css.preprocessorOptions**: CSS 预处理器配置（常用于注入全局 SCSS 变量/Mixin）。

### 2. 开发服务器 (Server Options)
- **server.host**: 设置为 `0.0.0.0` 可让局域网其他设备访问。
- **server.port**: 指定开发端口。
- **server.proxy**: 代理配置，解决开发环境接口跨域问题。
  - 示例：`'/api': { target: 'http://backend.com', changeOrigin: true }`。

### 3. 构建配置 (Build Options)
- **build.outDir**: 输出目录，默认为 `dist`。
- **build.target**: 浏览器兼容目标（如 `es2015`, `chrome80`）。
- **build.sourcemap**: 是否生成 sourcemap，用于生产环境排查报错。
- **build.rollupOptions**: 底层 Rollup 配置。
  - **output.manualChunks**: 配置代码分割策略（将第三方包打入 vendor 或拆分）。

### 4. 依赖优化 (Dep Optimization)
- **optimizeDeps**: 依赖预构建配置。常用于强制包含某些 CommonJS 依赖（`include`）或排除不需要预构建的包（`exclude`）。

