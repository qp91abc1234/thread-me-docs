---
theme: z-blue
---

## 问题：非 TypeScript 文件不会拷贝到 dist 目录

NestJS 使用 TypeScript 编译器对代码进行编译，编译后的文件输出到 `dist` 路径，然后开发服务器也是从 `dist` 路径下启动。因为 TypeScript 编译器不能拷贝非 TypeScript 文件，因此 dist 路径下没有非 TypeScript 文件。

## 解决方案：assets 配置

在 `nest-cli.json` 中配置 `assets` 选项：

```json
{
  "compilerOptions": {
    "assets": [
      {
        "include": "../public/**/*",
        "outDir": "dist/public"
      }
    ],
    "watchAssets": true,
    "deleteOutDir": true
  }
}
```

**作用：**
- NestJS CLI 在编译后额外复制非 `.ts` 文件到 `dist/`
- `watchAssets: true` 表示在开发模式下监听这些文件的变化
- 确保运行时可以访问到这些文件

## 配置说明

### assets 配置方式

**方式 1：字符串模式（简单）**
```json
{
  "assets": [
    "**/*.yml",        // 所有 .yml 文件
    "public/**/*",     // public 目录下所有文件
    "config/*.json"    // config 目录下的 .json 文件
  ]
}
```

**方式 2：对象模式（精细控制）**
```json
{
  "assets": [
    {
      "include": "../public/**/*",  // 包含的文件模式
      "outDir": "dist/public",      // 输出目录
      "exclude": "**/*.tmp"         // 排除的文件（可选）
    }
  ]
}
```

### 路径说明

- 路径相对于 `sourceRoot`（通常是 `src`）
- `"public/**/*"` 表示 `src/public/**/*`
- 如果文件在项目根目录，需要使用 `../` 向上查找

## 总结

1. **TypeScript 编译器**只处理 `.ts` 文件，编译输出到 `dist/`
2. **开发服务器**运行 `dist/` 下的编译后代码
3. **非 `.ts` 文件**需要 `assets` 配置来复制到 `dist/`
4. **这样运行时**才能访问到这些文件

