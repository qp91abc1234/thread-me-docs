## HTTP 缓存

- **缓存介质**
  - 资源最终是缓存到**内存还是磁盘**，完全由 **浏览器自身策略** 决定，**无法通过响应头精确指定**。

## gzip

- **vite-plugin-compression**
  - 插件默认只对 **JS / JSON / CSS 等文本资源**做 gzip 压缩。
  - **为什么默认不压图片？**
    - 图片（JPEG/PNG/WebP 等）本身已经做过**有损/无损压缩**；
    - 再用 gzip 压一次，文件体积：
      - 要么几乎不变，
      - 要么甚至稍微变大；
    - 但会额外增加：
      - 服务器的 **CPU 压缩开销**，
      - 浏览器的 **CPU 解压开销**。
    - 综合收益很低，所以默认不对图片再做 gzip。

- **Accept-Encoding**
  - `Accept-Encoding` 是 **浏览器自动添加** 的请求头，
  - 表示「浏览器可以接受哪些压缩格式」（如 `gzip` / `br` / `deflate`）。

- **Nginx gzip 静态 + 动态**
  - 在同时开启 `gzip_static on;` 和 `gzip on;` 的情况下：
    - 若请求的文件存在对应的 `.gz` 版本：
      - **直接返回该 `.gz` 文件**（静态 gzip），不再现场压缩；
    - 若不存在 `.gz`：
      - 且浏览器 `Accept-Encoding` 支持 gzip，
      - 则由 Nginx 对源文件进行 **动态 gzip 压缩** 后返回。