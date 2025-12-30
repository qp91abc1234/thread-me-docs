---
theme: z-blue
---

## 一、 Prisma Schema 数据模型 (Data Model)

Prisma 数据模型（Model）直接映射到数据库中的**表（Table）**或**集合（Collection）**。它是所有数据操作的基础：
1.  **作为数据库表结构的蓝图**（Prisma Migrate 会依据它来生成 SQL 并创建表）。
2.  **作为 TypeScript 类型定义的来源**（Prisma Client 会依据它生成对应的 TS 类型）。
3.  **作为数据库操作 API 的依据**（Prisma Client 会依据它生成增删改查的方法）。

```prisma
enum Role {
  USER
  ADMIN
}

model User {
  id        Int      @id @default(autoincrement()) // 主键，自增 ID
  email     String   @unique                       // 唯一索引
  name      String?                                // 可选字段
  role      Role     @default(USER)                // 枚举类型，默认值
  tags      String[]                               // 数组类型，不支持可选 ? (默认为空列表)
  createdAt DateTime @default(now())               // 创建时间，默认当前时间
}
```

### 1.1 基础定义

*   **字段类型**：
    *   `String`：字符串
    *   `Int`：整数
    *   `Boolean`：布尔值
    *   `DateTime`：日期时间
    *   `Json`：JSON数据
    *   `Bytes`：二进制
    *   `Decimal`：高精度数值
    *   `Float`：浮点数
*   **修饰符**：
    *   `?`：可选字段（nullable）。
    *   `[]`：数组。**注意**：在 Prisma 中，数组类型有两种用途：
        *   **关系字段**：用于定义一对多或多对多关系（如 `posts Post[]`）。
        *   **标量字段**：作为表的实际列类型（如 `tags String[]`），MySQL **不支持**，Prisma 会报错。如需在 MySQL 中存储数组数据，请使用 `Json` 类型（如 `tags Json`）。
        *   数组类型不支持可选 `?`，默认为空数组 `[]`。

### 1.2 常用属性装饰器

*   `@id`：主键。
*   `@default(...)`：默认值（`autoincrement()`, `now()`, `uuid()`, `cuid()`）。
*   `@unique`：唯一约束。
*   `@updatedAt`：自动更新时间戳。每次更新记录时，自动更新此字段为当前时间。
*   `@map("db_column_name")`：**字段映射**。在代码中使用驼峰（`firstName`），数据库中使用蛇形（`first_name`）。
*   `@@map("db_table_name")`：**表名映射**。重命名生成的数据库表名。

在实际开发中，数据库表和字段通常使用 **snake_case** (蛇形命名)，而 JavaScript/TypeScript 代码推荐使用 **camelCase** (驼峰命名)。`@map` 和 `@@map` 可以完美解决这个问题。

```prisma
model UserInfo {
  id        Int      @id @default(autoincrement())
  firstName String   @map("first_name") // 代码中用 user.firstName，数据库列名为 first_name
  lastName  String   @map("last_name")  // 代码中用 user.lastName，数据库列名为 last_name
  email     String   @unique            // 单字段唯一索引：保证 email 不重复
  updatedAt DateTime @updatedAt         // 每次更新记录时，自动更新此字段为当前时间

  @@map("users_info") // 代码中模型名为 UserInfo，数据库表名为 users_info
  @@unique([firstName, lastName]) // 复合唯一索引：firstName + lastName 的组合必须唯一
}
```

## 二、 Prisma Schema 关系建模 (Relations)

Prisma 的关系定义非常直观。

### 2.1 1:1 (一对一)
例如：用户 (User) 拥有一个个人资料 (Profile)。

```prisma
model User {
  id      Int      @id @default(autoincrement())
  profile Profile? // 关联字段，可选，因为创建用户时可能还没有 Profile
}

model Profile {
  id     Int  @id @default(autoincrement())
  userId Int  @unique // 必须是唯一索引，保证一对一
  user   User @relation(fields: [userId], references: [id])
}
```

### 2.2 1:n (一对多)
例如：用户 (User) 拥有多篇文章 (Post)。

```prisma
model User {
  id    Int    @id @default(autoincrement())
  posts Post[] // 关系字段：不用存到数据库，仅用于 Client 导航
}

model Post {
  id       Int  @id @default(autoincrement())
  authorId Int  // 外键字段 (实际存在数据库中)
  author   User @relation(fields: [authorId], references: [id]) // 关系定义
}
```

### 2.3 m:n (多对多)

Prisma 支持两种多对多关系实现方式：**隐式多对多**和**显式多对多**。

#### 2.3.1 隐式多对多（Implicit Many-to-Many）

Prisma 会自动创建并维护中间表，无需手动定义。外键约束默认为 `CASCADE`（级联删除）。

**适用场景**：
- 中间表不需要额外字段（如创建时间、备注等）
- 关系结构简单，无需自定义中间表逻辑

```prisma
model Post {
  id   Int   @id @default(autoincrement())
  tags Tag[] // 多对多关系：一篇文章可以有多个标签
}

model Tag {
  id    Int    @id @default(autoincrement())
  posts Post[] // 多对多关系：一个标签可以关联多篇文章
}
```

**说明**：
- Prisma 会自动创建 `_PostToTag` 中间表（命名规则：`_Model1ToModel2`）
- 中间表包含 `A` 和 `B` 两个外键字段
- 关联查询时使用 `include: { tags: true }` 或 `select: { tags: true }`，返回的是目标对象数组（如 `Tag[]`），无需手动处理中间表

#### 2.3.2 显式多对多（Explicit Many-to-Many）

手动定义中间表模型，可以添加额外字段和业务逻辑。

**适用场景**：
- 需要在中间表存储额外信息（如关联创建时间、关联状态等）
- 需要对中间表进行查询、更新等操作
- 需要自定义中间表的索引、约束等

```prisma
model Post {
  id      Int       @id @default(autoincrement())
  postTags PostTag[] // 指向中间表，返回 PostTag[] 对象数组
}

model Tag {
  id      Int       @id @default(autoincrement())
  postTags PostTag[] // 指向中间表，返回 PostTag[] 对象数组
}

// 显式中间表（Explicit Relation Table）
model PostTag {
  id        Int      @id @default(autoincrement())
  postId    Int
  tagId     Int
  createdAt DateTime @default(now()) // 额外字段：记录关联创建时间
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  tag       Tag      @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@unique([postId, tagId]) // 防止重复关联
  @@index([postId])
  @@index([tagId])
}
```

**说明**：
- 中间表是一个独立的模型，可以添加任意字段
- 查询时 `post.postTags` 返回的是 `PostTag[]`，要获取 `Tag[]` 需使用嵌套查询：`include: { postTags: { include: { tag: true } } }`
- 建议使用 `@@unique` 约束防止重复关联
- 建议添加索引优化查询性能

---

## 三、 Prisma Client 实战 (CRUD & Relations)

### 3.1 基础 CRUD

```typescript
// 1. Create (增)
const user = await prisma.user.create({
  data: { email: 'alice@prisma.io', name: 'Alice' },
});

// 2. Delete (删)
const deletedUser = await prisma.user.delete({
  where: { email: 'alice@prisma.io' },
});

// 3. Update (改)
const updatedUser = await prisma.user.update({
  where: { id: 1 },
  data: { name: 'Alice Wonderland' },
});

// 4. Read (查)
// 4.1 查找单个 (Find Unique / First)
const user = await prisma.user.findUnique({
  where: { id: 1 },
});

// 4.2 查找列表与分页 (Find Many with Pagination)
const users = await prisma.user.findMany({
  where: { name: { contains: 'Alice' } },
  skip: 0,    // 跳过前0条
  take: 10,   // 取10条
  orderBy: { createdAt: 'desc' }, // 按创建时间倒序
});

// 5. Upsert (更新或创建)
const result = await prisma.user.upsert({
  where: { email: 'alice@prisma.io' },
  update: { name: 'Alice' },
  create: { email: 'alice@prisma.io', name: 'Alice' },
});
```

### 3.2 关联查询 (Eager Loading)

Prisma 默认是 Lazy 的，不加载关联数据。我们需要使用 `include` 或 `select` 来进行预加载。这两个 API 经常容易混淆，我们可以用**“蛋糕模型”**来理解：

1.  **`select` 是“切块” (白名单模式)**：
    *   **主表：必须切块**。一旦用了 `select`，主表只返回显式选中的字段，其他字段不会返回。
    *   **从表（加料）：可加可不加**。如果加了从表，可以对从表选择“全要”或“只切一小块”。
2.  **`include` 是“加料” (增强模式)**：
    *   **主表：完全保留**。主表的所有标量字段（`id`, `name`...）默认全部返回。
    *   **从表（加料）：必须加**。在保留主表的基础上，额外挂载关联数据。可以对关联数据选择“全要”或“只切一小块”。

**对比图谱：**

| 需求场景 | 方案 | 结果 (User / Post) |
| :--- | :--- | :--- |
| **1. 我全都要** | `include: { posts: true }` | User ✅ / Post ✅ |
| **2. 我要User全字段，Post只要标题** | `include: { posts: { select: { title: true } } }` | User ✅ / Post(切块) |
| **3. User只要名字，Post全要** | `select: { name: true, posts: true }` | User(切块) / Post ✅ |
| **4. User只要名字，Post只要标题** | `select: { name: true, posts: { select: { title: true } } }` | User(切块) / Post(切块) |

**代码实战：**

```typescript
// 场景 1: 我全都要 (User全 + Post全)
const allData = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: true, // 加载所有文章字段
  },
});

// 场景 2: User全保留 + Post只取标题
// 最常用的场景：保留主对象详情，但关联列表只取关键信息
const userWithPostTitles = await prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: {
      select: { title: true }, // 对关联数据进行切块
    },
  },
});

// 场景 3: User只取名字 + Post全要
// 只要用了 select，User 就被切块了
const userNameWithFullPosts = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    name: true,  // User 显式切块
    posts: true, // Post 全要
  },
});

// 场景 4: User只取名字 + Post只取标题 (极致精简)
// 前后端接口优化常用，只返回 UI 需要的字段
const minimalData = await prisma.user.findUnique({
  where: { id: 1 },
  select: {
    name: true, // User 切块
    posts: {
      select: { title: true }, // Post 切块
    },
  },
});
```

### 3.3 嵌套操作 (Nested Operations)

Prisma 强大的特性之一是可以在一次操作中同时处理父子表数据。**在一个 `create` 或 `update` 里嵌套写入了多个表的数据，Prisma 会自动把它们包裹在一个事务里，这种叫隐式事务。如果任何一步失败，整个操作会回滚。**

#### 3.3.1 嵌套创建 (Nested Create)
```typescript
// 创建用户的同时，创建两篇文章
await prisma.user.create({
  data: {
    email: 'bob@prisma.io',
    posts: {
      create: [
        { title: 'My first post' },
        { title: 'My second post' },
      ],
    },
  },
});
```

#### 3.3.2 嵌套更新 (Nested Update)
可以在更新父记录时，同时对关联记录进行增、删、改。

```typescript
const updatedUser = await prisma.user.update({
  where: { id: 1 }, // 锁定父记录 (User)
  data: {
    name: 'Bob Updated', // 更新父记录字段

    // 处理关联字段 (Posts)
    posts: {
      // 1. 新增关联记录
      create: [
        { title: 'New Post via Nested Update' }
      ],

      // 2. 更新已有的关联记录
      update: {
        where: { id: 10 }, // 找到要更新的子记录
        data: { title: 'Updated Title' }
      },
      
      // 3. 删除已有的关联记录
      delete: {
        id: 11 // 找到要删除的子记录
      }
    }
  }
});
```

### 3.4 显式事务 (Explicit Transactions)

当嵌套操作无法满足需求（例如需要跨多个不相关的模型，或需要执行中间逻辑判断）时，使用显式事务。

**1. 顺序事务 ($transaction 数组模式)**
要么全部成功，要么全部失败。
```typescript
const [user, post] = await prisma.$transaction([
  prisma.user.create({ data: { ... } }),
  prisma.post.create({ data: { ... } }),
]);
```

**2. 交互式事务 (Interactive Transactions)**
允许在事务中执行逻辑判断。
```typescript
await prisma.$transaction(async (tx) => {
  // 1. 扣减余额
  const sender = await tx.user.update({
    where: { id: senderId },
    data: { balance: { decrement: amount } },
  });

  if (sender.balance < 0) {
    throw new Error('余额不足'); // 抛出错误，自动回滚
  }

  // 2. 增加接收方余额
  await tx.user.update({
    where: { id: receiverId },
    data: { balance: { increment: amount } },
  });
});
```

### 3.5 原生 SQL (Raw Database Access)
当 Prisma 的 API 无法满足需求（如复杂的报表统计）时，使用原生 SQL。

```typescript
// 查询：返回强类型数组（类型参数仅用于 TypeScript 类型推断，不会进行运行时验证）
const result = await prisma.$queryRaw<User[]>`SELECT * FROM User WHERE age > ${20}`;

// 执行：返回受影响行数
const count = await prisma.$executeRaw`UPDATE User SET active = true WHERE age > ${18}`;
```

---

## 四、 常用命令速查 (Prisma CLI)

### 4.1 反向工程 (`db pull`)

*   **指令**：`npx prisma db pull`
*   **功能定位**：
    *   从现有数据库生成 Schema。
*   **适用场景**：
    *   已有数据库项目接入 Prisma。
    *   手动修改了数据库结构，需要同步回 `schema.prisma`。

### 4.2 开发迁移 (`migrate dev`)

*   **指令**：`npx prisma migrate dev [--name <migration_name>]`
*   **适用场景**：
    *   在开发阶段修改了 schema.prisma，需要生成 SQL 迁移文件以记录变更历史并同步数据库时使用
*   **核心动作**：
    1.  **比对差异与生成迁移**：
        *   **场景 A（常规开发）**：有迁移历史。借助影子数据库，比对 `schema.prisma` 与现有迁移记录的差异，生成增量 SQL 迁移文件。
        *   **场景 B（项目初始化）**：无迁移历史且数据库为空。直接基于 Schema 生成完整的初始建表 SQL 迁移文件。
        *   **场景 C（漂移/冲突）**：无迁移历史但数据库有内容或有迁移历史但数据库被手动篡改，Prisma 会提示**重置数据库**（清空数据）。
            *   若同意重置：数据库被清空，根据有无迁移历史决定**回到场景 A**或**回到场景 B**。
            *   若拒绝重置：命令终止。
    2.  执行 SQL，更新开发数据库。
*   **常用参数**：
    *   `--name`: 指定迁移文件的名称（如 `init_users`）。
        *   **建议始终加上此参数**：如果不加，在自动化脚本（如 CI/CD）中运行时，CLI 会进入交互模式询问迁移名称，导致进程卡死。
        *   **即使写死名称也能工作**：即便将名称固定写死为 `update`，Prisma 依然会自动在文件名前加上唯一的时间戳前缀（如 `202310271230_update`），因此不会发生文件名冲突，且能保证自动化流程顺畅执行。

### 4.3 原型同步 (`db push`)

*   **指令**：`npx prisma db push --accept-data-loss`
*   **功能定位**：
    *   **项目初期原型开发**：在 Schema 频繁变动且尚未定稿的阶段，快速同步结构，避免生成大量无意义的迁移文件。
    *   **注意**：
        *   **无历史记录**：不生成 migration 文件。
        *   **数据丢失风险**：若涉及破坏性变更（如删列），需显式确认或使用参数强制执行，会导致数据丢失。
*   **核心动作**：
    1.  **直接比对与同步**：不依赖迁移文件，直接比对 `schema.prisma` 与数据库当前结构的差异。
    2.  **执行变更**：
        *   **无损变更**（如新增字段）：直接在数据库执行 SQL。
        *   **破坏性变更**（如删除字段、修改无法兼容的类型）：Prisma 会报错拦截，**必须**添加 `--accept-data-loss` 参数才会执行操作（此时会丢弃对应列的数据）。

### 4.4 客户端生成 (`generate`)

*   **指令**：`npx prisma generate`
*   **功能定位**：
    *   根据 schema.prisma 生成对应的 TypeScript 类型定义和 Prisma Client 客户端代码。
*   **适用场景**：
    *   Schema 变更后的代码同步，不同步数据库。

### 4.5 生产部署 (`migrate deploy`)

*   **指令**：`npx prisma migrate deploy`
*   **功能定位**：
    *   代码部署到生产/测试环境后，更新目标数据库结构。
    *   **注意**：此命令**不**会比对差异生成新的迁移文件，仅执行 `migrations/` 目录下尚未执行的 SQL。
*   **核心动作**：
    1.  检查 `_prisma_migrations` 表，找出未应用的迁移。
    2.  按顺序执行 SQL 迁移文件。

### 4.6 数据浏览器 (`studio`)

*   **指令**：`npx prisma studio`
*   **功能定位**：
    *   官方提供的图形化管理界面。
*   **适用场景**：
    *   查看、搜索、编辑数据库中的数据。
    *   代替 DBeaver / Navicat 进行简单的 CRUD 操作。
