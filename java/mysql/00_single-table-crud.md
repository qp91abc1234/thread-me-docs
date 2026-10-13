## SQL 的逻辑处理顺序

为了理解 SQL 的语义，可以先记住下面的简化顺序：

> 这是用于理解 SQL 语义的逻辑顺序，不代表 MySQL 实际采用的物理执行步骤。

```text
FROM
→ JOIN / ON
→ WHERE
→ GROUP BY
→ HAVING
→ SELECT
→ DISTINCT
→ ORDER BY
→ LIMIT
```

各阶段的作用如下：

1. `FROM`：确定数据来源。
2. `JOIN / ON`：连接其他表，并根据连接条件组合数据。
3. `WHERE`：过滤分组前的原始数据。
4. `GROUP BY`：将过滤后的数据分组。
5. `HAVING`：过滤分组后的结果。
6. `SELECT`：计算并选择需要返回的字段。
7. `DISTINCT`：去除重复的查询结果。
8. `ORDER BY`：对结果排序。
9. `LIMIT`：截取最终需要返回的数据。

## MyBatis-Plus 分层使用原则

```text
简单单表 CRUD
→ 优先使用 Service 通用方法

简单动态条件
→ 使用 Service 的 lambdaQuery / lambdaUpdate

多表、聚合、复杂 SQL
→ 在 Mapper 中定义 SQL，由 Service 调用

Controller
→ 通常只调用 Service，不直接调用 Mapper
```

## `NULL` 的正确判断

### 1. 介绍 `NULL`

`NULL` 表示某个字段的值未知、缺失或尚未设置。

它不等于以下任何一种值：

```text
数字 0
空字符串 ''
字符串 'null'
布尔值 false
```

### 2. MyBatis-Plus 判断 `NULL` 的写法

查询从未登录过的用户：

```java
List<SysUser> users = lambdaQuery()
        .isNull(SysUser::getLastLoginAt)
        .list();
```

对应 SQL：

```sql
WHERE last_login_at IS NULL
```

查询已经登录过的用户：

```java
List<SysUser> users = lambdaQuery()
        .isNotNull(SysUser::getLastLoginAt)
        .list();
```

对应 SQL：

```sql
WHERE last_login_at IS NOT NULL
```



### 3. 聚合函数与 `NULL`

大多数聚合函数会忽略字段中的 `NULL`，但 `COUNT(*)` 统计的是记录行数，不会因为某个字段为 `NULL` 而忽略该行。

假设存在以下数据：


| id  | username | age    |
| --- | -------- | ------ |
| 1   | user_a   | 20     |
| 2   | user_b   | 30     |
| 3   | user_c   | `NULL` |


执行：

```sql
SELECT
    COUNT(*) AS row_count,
    COUNT(age) AS age_count,
    SUM(age) AS age_sum,
    AVG(age) AS age_avg,
    MAX(age) AS age_max,
    MIN(age) AS age_min
FROM sys_user;
```

结果为：


| 聚合表达式        | 结果  | 说明                                   |
| ------------ | --- | ------------------------------------ |
| `COUNT(*)`   | 3   | 统计所有记录                               |
| `COUNT(age)` | 2   | 只统计 `age` 非 `NULL` 的记录               |
| `SUM(age)`   | 50  | 忽略 `NULL` 后求和                        |
| `AVG(age)`   | 25  | 按 `(20 + 30) / 2` 计算，不会把 `NULL` 当成 0 |
| `MAX(age)`   | 30  | 忽略 `NULL` 后取最大值                      |
| `MIN(age)`   | 20  | 忽略 `NULL` 后取最小值                      |


如果参与计算的字段值全部为 `NULL`，结果也需要特别注意：

```text
COUNT(字段)          → 0
SUM / AVG / MAX / MIN → NULL
```



### 4. `NULL` 的比较与 `UNKNOWN`

SQL 的条件判断存在三种结果：

```text
TRUE
FALSE
UNKNOWN
```

任何普通比较只要涉及 `NULL`，结果通常都是 `UNKNOWN`：


| 表达式            | 结果        |
| -------------- | --------- |
| `1 = 1`        | `TRUE`    |
| `1 = 2`        | `FALSE`   |
| `1 = NULL`     | `UNKNOWN` |
| `NULL = NULL`  | `UNKNOWN` |
| `NULL <> 1`    | `UNKNOWN` |
| `NULL <> NULL` | `UNKNOWN` |


`WHERE` 只保留判断结果为 `TRUE` 的记录，结果为 `FALSE` 或 `UNKNOWN` 的记录都会被过滤。

### 5. `<>` 遇到 `NULL`

`<>` 表示不等于，MySQL 也支持写成 `!=`：

假设表中存在三条记录：


| username | status |
| -------- | ------ |
| user_a   | `0`    |
| user_b   | `1`    |
| user_c   | `NULL` |


执行：

```sql
SELECT username, status
FROM sys_user
WHERE status <> 0;
```

结果为 `UNKNOWN`，而 `WHERE` 只保留结果为 `TRUE` 的记录。

业务角度来看，该被选出而未被选出。

### 6. `NOT IN` 遇到 `NULL`

如果字段本身为 `NULL`：

```sql
WHERE status NOT IN (0, 2)
```

可以近似展开为：

```sql
WHERE status <> 0
  AND status <> 2
```

但当 `status` 为 `NULL` 时，两个比较结果都是 `UNKNOWN`，因此这条记录会被 `WHERE` 过滤，不会出现在查询结果中。

业务角度来看，该被选出而未被选出。

## `ORDER BY`

> 业务排序字段不唯一时，追加主键作为最终排序条件，目的是让任意两条记录都能确定唯一的先后顺序。

如果排序字段不是唯一字段，多条记录就可能具有相同的排序值。

例如，下面三条记录的创建时间完全相同：


| id  | username | created_at          |
| --- | -------- | ------------------- |
| 101 | user_a   | 2026-10-12 10:00:00 |
| 102 | user_b   | 2026-10-12 10:00:00 |
| 103 | user_c   | 2026-10-12 10:00:00 |


如果只按照创建时间排序：

```sql
ORDER BY created_at DESC
```

MySQL 只能确定这三条记录属于同一个排序位置，无法根据当前条件确定它们之间的先后顺序。它们的实际返回顺序可能受到执行计划、索引选择和数据存储状态等因素影响。

普通查询中，这可能表现为相同数据每次返回的先后顺序不稳定；分页查询中，则可能造成：

```text
同一条记录出现在相邻两页；
某条记录在翻页时被遗漏；
相同条件下的列表顺序发生抖动。
```

因此，应追加一个唯一字段作为最终排序条件，通常使用主键：

```sql
SELECT id, username, created_at
FROM sys_user
ORDER BY created_at DESC, id DESC;
```

这样，即使 `created_at` 相同，也能继续通过唯一的 `id` 确定先后顺序，从而形成稳定排序。

MyBatis-Plus：

```java
List<SysUser> users = lambdaQuery()
        .orderByDesc(SysUser::getCreatedAt)
        .orderByDesc(SysUser::getId)
        .list();
```



## `LIMIT` 分页

### 1. `LIMIT` 介绍

`LIMIT` 用于限制查询结果返回的记录数量。

查询前 20 条用户记录：

```sql
SELECT id, username, created_at
FROM sys_user
ORDER BY id ASC
LIMIT 20;
```

分页时常用的语法是：

```sql
LIMIT offset, size
```

- `offset`：跳过多少条记录。
- `size`：最多返回多少条记录。

例如：

```sql
-- 第一页：跳过 0 条，返回 20 条
LIMIT 0, 20;

-- 第二页：跳过 20 条，返回 20 条
LIMIT 20, 20;
```

假设页码 `pageNo` 从 1 开始，每页数量为 `pageSize`，偏移量计算公式为：

```text
offset = (pageNo - 1) × pageSize
```

### 2. 分页必须要有排序

不推荐直接分页而不指定顺序：

```sql
SELECT id, username
FROM sys_user
LIMIT 0, 20;
```

没有 `ORDER BY` 时，MySQL 不保证记录的返回顺序。执行计划、索引选择或数据变化都可能改变结果顺序，导致翻页时出现记录重复或遗漏。

应该先形成稳定顺序，再截取当前页：

```sql
SELECT id, username, created_at
FROM sys_user
ORDER BY created_at DESC, id DESC
LIMIT 0, 20;
```

其中 `created_at` 可能重复，因此追加唯一的主键 `id` 作为最终排序条件。

```text
分页查询 = 稳定排序 + LIMIT
```

### 3. MyBatis-Plus 分页

MyBatis-Plus 使用 `Page` 对象描述当前页码和每页数量：

```java
Page<SysUser> pageRequest = new Page<>(pageNo, pageSize);

LambdaQueryWrapper<SysUser> wrapper =
        Wrappers.<SysUser>lambdaQuery()
                .eq(
                        status != null,
                        SysUser::getStatus,
                        status
                )
                .orderByDesc(SysUser::getCreatedAt)
                .orderByDesc(SysUser::getId);

IPage<SysUser> result = page(pageRequest, wrapper);
```

MyBatis-Plus 的页码通常从 `1` 开始，它会根据页码和每页数量计算 SQL 中的偏移量。

项目需要正确配置 MyBatis-Plus 分页插件，分页 SQL 才会被正确改写。

### 4. 分页结果包含什么

`IPage<SysUser>` 中常用的数据包括：

```java
// 当前页的数据
List<SysUser> records = result.getRecords();

// 满足查询条件的总记录数
long total = result.getTotal();

// 当前页码
long current = result.getCurrent();

// 每页数量
long size = result.getSize();

// 总页数
long pages = result.getPages();
```

### 5. 数据量大、页码深时考虑游标分页

下面的深分页 SQL 虽然语法正确：

```sql
SELECT id, username
FROM sys_user
ORDER BY id ASC
LIMIT 100000, 20;
```

但 MySQL 仍然需要找到并跳过前面的 100000 条记录。偏移量越大，查询通常越慢。

如果业务不要求任意跳转页码，可以记录上一页最后一条数据的 ID：

```sql
SELECT id, username
FROM sys_user
WHERE id > ?
ORDER BY id ASC
LIMIT 20;
```

MyBatis-Plus：

```java
List<SysUser> users = lambdaQuery()
        .gt(lastId != null, SysUser::getId, lastId)
        .orderByAsc(SysUser::getId)
        .last("LIMIT 20")
        .list();
```

这里的分页数量必须由后端控制，不能将未经校验的前端参数直接拼接到 `last()` 中。

```text
普通后台分页：Page + offset 分页；
数据量大、页码深：考虑游标分页；
游标分页性能更稳定，但不适合任意跳转到指定页码。
```

## MyBatis-Plus 逻辑删除

### 1. `@TableLogic`：删除改标记、查询自动过滤、更新自动过滤

实体的逻辑删除字段可以使用 `@TableLogic` 标记：

```java
@TableLogic
private Integer deleted;
```

通常约定：

```text
deleted = 0：未删除
deleted = 1：已删除
```

配置逻辑删除后，MyBatis-Plus 提供的通用方法会自动处理逻辑删除条件。

#### 删除改为标记

调用：

```java
userService.removeById(userId);
```

实际执行的 SQL 类似于：

```sql
UPDATE sys_user
SET deleted = 1
WHERE id = ?
  AND deleted = 0;
```

数据没有被物理移除，只是被标记为已删除。

#### 查询自动过滤

调用：

```java
List<SysUser> users = userService.lambdaQuery()
        .eq(SysUser::getStatus, 1)
        .list();
```

生成的 SQL 类似于：

```sql
SELECT *
FROM sys_user
WHERE status = 1
  AND deleted = 0;
```

通过 MyBatis-Plus 通用方法查询时，已逻辑删除的数据会被自动过滤。

#### 更新自动过滤

调用：

```java
boolean success = userService.lambdaUpdate()
        .eq(SysUser::getId, userId)
        .set(SysUser::getStatus, 0)
        .update();
```

生成的 SQL 类似于：

```sql
UPDATE sys_user
SET status = 0
WHERE id = ?
  AND deleted = 0;
```

已经逻辑删除的数据不会被 MyBatis-Plus 的普通更新方法再次更新。

可以将 `@TableLogic` 的核心行为记成：

```text
删除：DELETE 改为 UPDATE 删除标记；
查询：自动追加未删除条件；
更新：自动追加未删除条件。
```

### 2. 自定义 SQL 必须关注逻辑删除条件

`@TableLogic` 主要作用于 MyBatis-Plus 生成的通用 SQL。自己编写的 Mapper XML 或注解 SQL，需要主动添加逻辑删除条件。

例如：

```xml
<select id="selectUserPermissions"
        resultType="com.example.rbac.vo.UserPermissionVO">
    SELECT
        u.id,
        u.username,
        p.permission_code
    FROM sys_user u
    JOIN sys_user_role ur
        ON ur.user_id = u.id
    JOIN sys_role_permission rp
        ON rp.role_id = ur.role_id
    JOIN sys_permission p
        ON p.id = rp.permission_id
    WHERE u.id = #{userId}
      AND u.deleted = 0
      AND p.deleted = 0
</select>
```

如果遗漏：

```sql
AND u.deleted = 0
```

自定义 SQL 就可能重新查询出已经逻辑删除的用户。

