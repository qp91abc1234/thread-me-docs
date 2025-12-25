---
theme: z-blue
---

## 一、 Prisma 核心概念

### 1.1 Prisma 是什么
Prisma 是下一代 Node.js 和 TypeScript ORM。与传统的 ORM（如 TypeORM）不同，Prisma 不使用类（Classes）来映射数据库表，而是使用自定义的 Schema 语言来定义模型，并生成 **类型安全（Type-safe）** 的查询构建器（Client）。

> 本文基于 Prisma7

### 1.2 核心组件
Prisma 生态系统由三个主要工具组成：

1.  **Prisma Client**: 基于你的 Schema 自动生成的类型安全查询构建器，供 Node.js 和 TypeScript 调用。
2.  **Prisma Migrate**: 声明式的数据建模与迁移系统。
3.  **Prisma Studio**: 现代化的数据库 GUI，用于浏览和管理数据（类似 DBeaver/Navicat 的 Web 精简版）。

---

## 二、 NestJS 集成与初始化

### 2.1 项目初始化

在现有的 NestJS 项目中安装 Prisma CLI 和 Client：

```bash
# 安装 CLI (开发依赖)
pnpm add -D prisma

# 安装 Client (运行时依赖)
pnpm add @prisma/client

# 安装驱动程序适配器
pnpm add @prisma/adapter-mariadb

# 安装 dotenv 用于加载环境变量
pnpm add -D dotenv

# 初始化 (生成 prisma 文件夹和 .env, prisma.config.ts)
npx prisma init
```

### 2.2 核心文件详解

Prisma 7 引入了 **配置即代码** 的理念，核心配置分散在以下文件中：

*   **`prisma.config.ts`**：Prisma 的主配置文件，负责指定 Schema 文件位置、配置迁移文件生成路径以及配置数据源连接。
*   **`prisma/schema.prisma`**：指定数据源类型（Provider）、生成器（Generator）和定义数据模型（Models）。
*   **`.env`**：存储环境变量（如 `DATABASE_URL`）。Prisma CLI 默认只读取此文件，但可以通过系统环境变量覆盖，或在 `prisma.config.ts` 中自定义加载逻辑。

**1. prisma.config.ts 示例**：
```typescript
import 'dotenv/config';
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  // 1. 指定 schema 文件位置
  schema: 'prisma/schema.prisma',
  
  // 2. 配置迁移文件生成路径
  migrations: {
    path: 'prisma/migrations',
  },
  
  // 3. 配置数据源连接
  // 这里使用 env() 函数读取环境变量，实现了配置与代码的分离
  datasource: {
    url: env('DATABASE_URL'),
  },
});
```

**2. prisma/schema.prisma 示例**：
```prisma
// 指定生成器
generator client {
  provider = "prisma-client"
  output   = "./generated"
}

// 指定数据源类型 (注意：此处不再配置 url)
datasource db {
  provider = "mysql"
}

// 定义 User 模型
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}
```

### 2.3 NestJS 模块化集成 (PrismaService)

为了在 NestJS 中高效使用，我们需要创建一个全局 Service 来管理连接生命周期，并在业务模块中注入使用。

**步骤 1：创建全局 PrismaService**

**src/prisma/prisma.service.ts**:
```typescript
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';
import { PrismaMariaDb } from '@prisma/adapter-mariadb';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    super({
      adapter: new PrismaMariaDb(process.env.DATABASE_URL)),
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

**src/prisma/prisma.module.ts**:
```typescript
import { Global, Module } from '@nestjs/common';
import { PrismaService } from './prisma.service';

@Global() // 设为全局模块，避免到处 import
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

**步骤 2：在业务模块中使用**

假设有一个 `UsersModule`，你只需要在 Service 的构造函数中注入 `PrismaService` 即可。

**src/users/users.service.ts**:
```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { User, Prisma } from '@prisma/client'; // 导入生成的类型

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }

  async findAll() {
    return this.prisma.user.findMany();
  }
}
```

---

## 三、 Prisma 核心链路

1.  **配置阶段 (`prisma.config.ts`)**：
    *   它是总指挥。当你运行 `prisma` 命令时，它首先被加载。
    *   它告诉 CLI：去哪找 Schema 定义 (`schema.prisma`)，去哪找迁移文件 (`migrations`)，以及去哪找数据库连接串 (`.env` -> `DATABASE_URL`)。

2.  **反向工程阶段 (Introspection, 可选)**：
    *   **命令**：`npx prisma db pull`
    *   **场景**：当你有现存的数据库，或者有人直接修改了数据库结构时。
    *   **作用**：Prisma 读取数据库的 Schema（表、列、索引），并将它们 **反向生成/覆盖** 到 `prisma/schema.prisma` 文件中，确保你的 Schema 定义与真实数据库保持同步。

3.  **迁移同步阶段 (Migration/Sync)**：
    *   当你运行 `npx prisma migrate dev` 或 `npx prisma db push` 时，Prisma 会计算差异并同步到数据库。
    *   **`migrate dev`**：
        *   **差异计算**：比对 `schema.prisma` 与**迁移历史**（借助影子数据库），生成增量 SQL 迁移文件。
            *   **场景 A（常规开发）**：有迁移历史，比对 Schema 与现有迁移记录的差异，生成增量 SQL 迁移文件。
            *   **场景 B（项目初始化）**：无迁移历史且数据库为空，直接基于 Schema 生成完整的初始建表 SQL 迁移文件。
            *   **场景 C（漂移/冲突）**：无迁移历史但数据库有内容，或有迁移历史但数据库被手动篡改，Prisma 会提示**重置数据库**（清空数据）。
                *   若同意重置：数据库被清空，根据有无迁移历史决定**回到场景 A**或**回到场景 B**。
                *   若拒绝重置：命令终止。
        *   **执行变更**：将新生成的 SQL 迁移文件应用到数据库。
    *   **`db push`**：
        *   **比对与应用**：比对 `schema.prisma` 与**数据库当前结构**。忽略迁移历史，根据差异内容，直接将变更应用到数据库。
        *   **变更处理**：
            *   **无损变更**（如新增字段）：直接在数据库执行 SQL。
            *   **破坏性变更**（如删除字段、修改无法兼容的类型）：Prisma 会报错拦截，**必须**添加 `--accept-data-loss` 参数才会执行操作（此时会丢弃对应列的数据）。

4.  **生成阶段 (CodeGen)**：
    *   当你运行 `npx prisma generate` 时，Prisma 读取 `schema.prisma` 中的 `model User { ... }`。
    *   它将这些模型定义“翻译”成 TypeScript 类型定义 (`User`, `UserCreateInput`) 和 JavaScript 查询逻辑。
    *   生成的代码被写入到 `client 生成器` 配置的 `ouput` 目录下。
    *   **关键点**：这就是为什么每次修改 `schema.prisma` 后**必须**重新运行 `generate`，否则你的代码里拿不到新字段的提示。

5.  **运行阶段 (Runtime)**：
    *   当你执行 `this.prisma.user.findMany()` 时，`PrismaService` 实例（已在应用启动时连接数据库）会将 JS 对象转换成 SQL 语句发送给数据库，并将结果反序列化回 JS 对象。

## 四、 开发工作流集成

1.  **代码自动生成 (`postinstall`)**：
    *   在 `package.json` 中配置 `"postinstall": "prisma generate"`。
    *   **作用**：每次你（或同事、CI/CD）执行 `npm install` 安装依赖后，会自动触发 `prisma generate`。这样确保生成代码永远是最新的，你下载完项目就能直接跑，不用担心缺少 Prisma 类型代码。

2.  **修改表结构的流程**：
    *   **标准模式 (`migrate dev`)**：
        *   **动作**：运行 `npx prisma migrate dev` -> 生成新的 SQL 迁移文件 -> 执行 SQL 更新数据库 -> 运行 `prisma generate` 重新生成 Client 代码。
        *   **结果**：数据库表结构变更，且生成代码更新，NestJS 热重载触发，IDE 获得新字段提示。
    *   **Sync 模式 (`db push`)**：
        *   **动作**：运行 `npx prisma db push` -> 直接更新数据库结构（无迁移文件） -> 运行 `prisma generate` 重新生成 Client 代码。
        *   **场景**：适用于快速原型开发。

3.  **运行时连接 (`PrismaService`)**：
    *   我们在 `2.3` 节创建的 `PrismaService` 实现了 `OnModuleInit` 接口。
    *   **作用**：当 NestJS 应用启动 (`npm run start:dev`) 时，`PrismaService` 会自动调用 `$connect()` 建立数据库连接池。应用停止时自动断开。
    *   你不需要在每个 Controller 里手动 connect/disconnect，直接注入 Service 使用即可。

---

## 五、 深入理解：`migrate dev` 背后的黑盒机制

Prisma Migrate 的核心逻辑是基于 **Shadow Database（影子数据库）** 来计算差异的。

当你运行 `prisma migrate dev` 时，Prisma 会在后台偷偷做这件事：

1.  **创建影子库**：在后台临时创建一个空的数据库（Shadow DB）。
2.  **重放历史**：把本地 `prisma/migrations` 文件夹里所有的 SQL 文件，按顺序在这个影子库里执行一遍。执行完后，影子库的状态就是 “上一次迁移后的状态” (B)。
3.  **计算差异**：拿你现在的 `schema.prisma` (A) 和这个影子库 (B) 对比。
4.  **生成新迁移**：差异部分生成新的 SQL 文件，放入 `prisma/migrations`。
5.  **清理**：删掉影子库。

**结论：**

*   Prisma 信任的是你 **本地的文件系统 (`prisma/migrations`)** 作为历史记录的依据。
*   如果你把本地的 `migrations` 文件夹删了，Prisma 就失忆了，下次它就会认为你是从零开始，生成全量迁移。
*   所以，**`prisma/migrations` 文件夹必须提交到 Git**，它是项目源代码的重要组成部分。

