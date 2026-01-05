## Vite/Rollup 插件钩子

### 1. 初始化阶段 (Initialization)
插件初始化，读取与修正配置。
- **config** (Vite Specific | Dev + Build): 在解析 Vite 配置前调用，可**修改配置**。
- **configResolved** (Vite Specific | Dev + Build): 在解析 Vite 配置后调用，可**读取最终配置**。
- **options** (Dev + Build): 在构建启动前调用，用于**操作 Rollup 输入选项**。
- **configureServer** (Vite Specific | Dev Only): 用于**配置开发服务器**（如添加中间件），在 Server 启动前调用。
- **buildStart** (Dev + Build): 构建正式开始时调用。

### 2. 模块转换阶段 (Module Resolution & Transformation)
核心处理流程，对每个导入的模块循环执行。

- **resolveId** (Dev + Build): 解析并返回模块 ID（即模块的唯一标识符）。
  - **机制**：一旦某个插件返回结果，后续插件的该钩子不再执行（排他性）。
  - **说明**：
    - 对于真实文件：模块 ID 通常是文件的**绝对路径**。
    - 对于虚拟模块：模块 ID 是自定义的唯一字符串（通常以 `\0` 开头）。
  - **参数**：
    - `source`：被导入的模块路径（如 `"./foo.js"` 或 `"vue"`）。
    - `importer`：导入者（即引用该模块的文件）的绝对路径。
  - **返回值**：
    - `string`：解析后的模块绝对路径（或虚拟模块 ID）。
    - `object`：包含 `{ id, external }` 等配置的对象。
      - `id`：模块 ID。
      - `external`：`true` 表示将其视为外部依赖（不打包）。
    - `null` / `undefined`：无法解析，交给下一个插件处理。

- **load** (Dev + Build): 加载模块内容。
  - **机制**：一旦某个插件返回结果，后续插件的该钩子不再执行（排他性）。
  - **参数**：
    - `id`：模块 ID。
  - **返回值**：
    - `string`：模块的源代码。
    - `object`：包含 `{ code, map }` 的对象（支持 SourceMap）。
    - `null` / `undefined`：交给下一个插件处理（通常是读取磁盘文件）。

- **transform** (Dev + Build): 转换模块代码。
  - **机制**：链式执行，上一个插件转换后的代码会传给下一个插件继续转换。
  - **参数**：
    - `code`：模块源代码。
    - `id`：模块 ID。
  - **返回值**：
    - `string`：转换后的代码。
    - `object`：包含 `{ code, map }` 的对象。
    - `null` / `undefined`：不转换。

### 3. HTML 处理阶段 (HTML Transformation)
处理入口 HTML 文件。
- **transformIndexHtml** (Vite Specific | Dev + Build): 转换 HTML 内容。
  - **机制**：链式执行，多个插件可以依次修改 HTML。
  - **参数**：
    - `html`：当前的 HTML 字符串。
    - `ctx`：上下文对象（包含 `filename`, `server` 等）。
  - **返回值**：
    - `string`：修改后的 HTML 字符串。
    - `object[]`：要注入的标签数组（Tag Descriptor）。
    - `object`：包含 `{ html, tags }` 的对象。

### 4. 构建结束与输出阶段 (Build End & Output Generation)
构建完成，生成产物并写入磁盘。
- **buildEnd** (Dev + Build): 构建逻辑结束时调用（无论成功或失败）。
- **outputOptions** (Build Only): 接收并可**修改 Rollup 输出选项**。
- **renderStart** (Build Only): 开始生成 Bundle 时调用。
- **renderChunk** (Build Only): 主要用于**修改 chunk**（对每个生成的 chunk 代码进行转换、压缩、混淆等）。
- **generateBundle** (Build Only): 在产物写入磁盘前调用，可**操作最终的 Asset/Chunk**。
  - **bundle**：构建产物的集合对象，包含所有输出的 Asset 和 Chunk
  - **Asset**：构建产物中的静态资源文件的对象化描述，包含 html、css、字体文件、json...
  - **Chunk**：构建产物中 JavaScript 代码模块的对象化描述
    - Module 是构建过程中对源代码模块的对象化描述
    - Chunk 由一个或多个 Module 经过编译、打包后生成
- **writeBundle** (Build Only): 在产物写入磁盘后调用。
- **closeBundle** (Build Only): 构建完全结束时调用。

### 5. 热更新 (HMR)
- **handleHotUpdate** (Vite Specific | Dev Only): 自定义 HMR 更新处理。
