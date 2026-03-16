## 特性

### 缓存

- **微信原生小程序缓存** 与 **内嵌 WebView（H5）的浏览器缓存** 不是同一套机制、也不共用存储空间，调试缓存相关问题时要区分这两者。

### 样式隔离 `styleIsolation`

- **小程序原生**
  - `isolated`：启用样式隔离，在自定义组件内外，使用 `class` 指定的样式互不影响。
- **uni-app**
  - `apply-shared`：页面 `wxss` 样式会影响到自定义组件，但自定义组件 `wxss` 中的样式不会反向影响页面。

### `src/static`

- `src/static`（或 `static`）下的资源，会在构建时**原样拷贝**到对应平台的产物目录中（例如小程序端的 `dist/build/mp-weixin/static`）。
- 小程序在审核 / 运行时，会把这些文件**计入包体积**（主包或对应分包），需要注意体积限制。

## 组件

### `image`

- **H5 环境**：`<image>` 不支持原生懒加载，需要自行封装（如 IntersectionObserver）。
- **小程序环境**：原生 `<image>` 支持懒加载能力。

### 组件封装

在 uni-app 中基于组件进行二次封装（尤其是面向小程序端时，如 `wd-img -> lazy-img`），需要注意：

1. 由于**组件节点的存在**，无法像 Web 那样简单地将组件外的属性透传到组件根节点。
2. 使用 `v-bind="$attrs"` / `v-bind="attrs"` 会在小程序编译链中导致报错：`v-bind="" is not supported`。

**因此**：基于现有组件进行二次封装时，应显式定义 props，并配合 `virtualHost` / `mergeVirtualHostAttributes` 等机制控制根节点样式与属性，而不是依赖 `$attrs` 批量透传。