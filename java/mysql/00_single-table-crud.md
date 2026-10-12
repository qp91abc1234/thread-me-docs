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

| id | username | age |
| ---: | --- | ---: |
| 1 | user_a | 20 |
| 2 | user_b | 30 |
| 3 | user_c | `NULL` |

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

| 聚合表达式 | 结果 | 说明 |
| --- | ---: | --- |
| `COUNT(*)` | 3 | 统计所有记录 |
| `COUNT(age)` | 2 | 只统计 `age` 非 `NULL` 的记录 |
| `SUM(age)` | 50 | 忽略 `NULL` 后求和 |
| `AVG(age)` | 25 | 按 `(20 + 30) / 2` 计算，不会把 `NULL` 当成 0 |
| `MAX(age)` | 30 | 忽略 `NULL` 后取最大值 |
| `MIN(age)` | 20 | 忽略 `NULL` 后取最小值 |

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

| 表达式 | 结果 |
| --- | --- |
| `1 = 1` | `TRUE` |
| `1 = 2` | `FALSE` |
| `1 = NULL` | `UNKNOWN` |
| `NULL = NULL` | `UNKNOWN` |
| `NULL <> 1` | `UNKNOWN` |
| `NULL <> NULL` | `UNKNOWN` |

`WHERE` 只保留判断结果为 `TRUE` 的记录，结果为 `FALSE` 或 `UNKNOWN` 的记录都会被过滤。

### 5. `<>` 遇到 `NULL`

`<>` 表示不等于，MySQL 也支持写成 `!=`：

假设表中存在三条记录：

| username | status |
| --- | ---: |
| user_a | `0` |
| user_b | `1` |
| user_c | `NULL` |

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

| id | username | created_at |
| ---: | --- | --- |
| 101 | user_a | 2026-10-12 10:00:00 |
| 102 | user_b | 2026-10-12 10:00:00 |
| 103 | user_c | 2026-10-12 10:00:00 |

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