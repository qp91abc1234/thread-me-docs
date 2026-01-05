## generateBundle
bundle 是构建产物的集合对象，包含所有输出的 Asset 和 Chunk
generateBundle 用于在产物已生成但写入磁盘前对其进行处理
    Asset 是构建产物中的静态资源文件的对象化描述, 包含 html，css，字符文件，json...
    Chunk 是构建产物中 JavaScript 代码模块的对象化描述
        Chunk 由一个或多个 Module 经过编译、打包后生成
        Module 是构建过程中对源代码模块的对象化描述