## html 转换插件
ejs 模板渲染
    transformIndexHtml
        将 html 模板字符串替换为数据，动态渲染 HTML 模板
HTML 压缩
    generateBundle
        通过 html-minifier-terser 在构建产物生成阶段对 HTML 进行压缩

## 图片压缩插件
generateBundle 钩子中遍历 bundle 对象拿到类型为图片的 Asset
通过压缩插件对其进行压缩

## 图片上传插件
generateBundle 钩子中遍历 bundle 对象拿到类型为图片的 Asset，对其进行上传
遍历产物中的 js，css 文件进行图片路径正则替换
删除 bundle 中图片类型的 Asset
