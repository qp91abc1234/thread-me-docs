## 联表查询

假设存在用户表和用户角色关系表。

`sys_user`：

| id | username |
| ---: | --- |
| 1 | admin |
| 2 | tom |
| 3 | jerry |

`sys_user_role`：

| user_id | role_id |
| ---: | ---: |
| 1 | 10 |
| 1 | 20 |
| 2 | 20 |

它们表达的关系是：

```text
admin → 拥有两个角色
tom   → 拥有一个角色
jerry → 没有角色
```

## `INNER JOIN`

`INNER JOIN` 只保留左右两张表能够根据 `ON` 条件匹配成功的数据。

```sql
SELECT
    u.id,
    u.username,
    ur.role_id
FROM sys_user u
INNER JOIN sys_user_role ur
    ON ur.user_id = u.id;
```

结果：

| id | username | role_id |
| ---: | --- | ---: |
| 1 | admin | 10 |
| 1 | admin | 20 |
| 2 | tom | 20 |

`jerry` 没有对应的角色关系，因此不会出现在结果中。

`INNER` 可以省略，下面两种写法含义相同：

```sql
INNER JOIN sys_user_role ur
    ON ur.user_id = u.id
```

```sql
JOIN sys_user_role ur
    ON ur.user_id = u.id
```

可以记成：

```text
INNER JOIN
→ 只返回连接成功的数据
```

## `LEFT JOIN`

`LEFT JOIN` 会保留左表的全部数据。右表能够匹配时返回右表字段，匹配不到时仍然保留左表记录。

```sql
SELECT
    u.id,
    u.username,
    ur.role_id
FROM sys_user u
LEFT JOIN sys_user_role ur
    ON ur.user_id = u.id;
```

这里：

```text
sys_user      → 左表
sys_user_role → 右表
```

结果：

| id | username | role_id |
| ---: | --- | ---: |
| 1 | admin | 10 |
| 1 | admin | 20 |
| 2 | tom | 20 |
| 3 | jerry | `NULL` |

可以记成：

```text
LEFT JOIN
→ 左表全部保留
→ 右表匹配成功就返回对应字段
→ 右表匹配失败就使用 NULL 补充右表字段
```

## 多次 `LEFT JOIN` 时，前一次连接失败不会中断后续连接

查询用户及其角色时，需要连续连接三张表：

```text
sys_user
→ sys_user_role
→ sys_role
```

```sql
SELECT
    u.id,
    u.username,
    ur.role_id,
    r.role_name
FROM sys_user u
LEFT JOIN sys_user_role ur
    ON ur.user_id = u.id
LEFT JOIN sys_role r
    ON r.id = ur.role_id;
```

对于没有角色的 `jerry`，第一次连接：

```sql
LEFT JOIN sys_user_role ur
    ON ur.user_id = u.id
```

无法找到匹配记录，但因为使用的是 `LEFT JOIN`，用户记录不会消失，而是得到一条中间结果：

```text
u.id       = 3
u.username = jerry
ur.user_id = NULL
ur.role_id = NULL
```

随后仍然会继续处理第二次连接：

```sql
LEFT JOIN sys_role r
    ON r.id = ur.role_id
```

此时连接条件实际相当于：

```sql
r.id = NULL
```

该条件无法匹配任何角色。因为第二次连接仍然是 `LEFT JOIN`，当前用户记录继续保留，`sys_role` 的字段也使用 `NULL` 补充：

```text
u.id        = 3
u.username  = jerry
ur.user_id  = NULL
ur.role_id  = NULL
r.id        = NULL
r.role_name = NULL
```

对于 `jerry`，整个连接过程是：

```text
第一次 LEFT JOIN：
u 中有 jerry，ur 中匹配不到
→ 保留 jerry，ur.* 补为 NULL

第二次 LEFT JOIN：
使用 ur.role_id = NULL 继续匹配 r.id
→ 匹配不到角色，仍然保留 jerry，r.* 继续补为 NULL
```

最终结果类似：

| id | username | role_id | role_name |
| ---: | --- | ---: | --- |
| 1 | admin | 10 | 超级管理员 |
| 1 | admin | 20 | 普通用户 |
| 2 | tom | 20 | 普通用户 |
| 3 | jerry | `NULL` | `NULL` |

因此，更准确的理解是：

```text
前一次 LEFT JOIN 失败
→ 不会删除左侧记录
→ 不会中断后续 JOIN
→ 后续 JOIN 仍然会继续处理
```

## `ON` 与 `WHERE` 的区别

可以先记住：

1. `ON`：决定右表哪些记录可以匹配。
2. `WHERE`：决定最终哪些结果行可以保留。

这个区别在 `LEFT JOIN` 中尤其重要。

假设需要查询所有用户，并且只关联启用状态的角色。即使用户没有启用角色，也要保留该用户。

将右表条件放在 `ON` 中：

```sql
SELECT
    u.id,
    u.username,
    r.id AS role_id,
    r.role_name
FROM sys_user u
LEFT JOIN sys_user_role ur
    ON ur.user_id = u.id
LEFT JOIN sys_role r
    ON r.id = ur.role_id
   AND r.status = 1
WHERE u.deleted = 0;
```

这里的：

```sql
ON r.id = ur.role_id
AND r.status = 1
```

表示只有启用角色可以参与右表匹配。如果匹配不到启用角色，左侧用户仍然保留，角色字段使用 `NULL` 补充。

如果把相同条件放到 `WHERE`：

```sql
SELECT
    u.id,
    u.username,
    r.id AS role_id,
    r.role_name
FROM sys_user u
LEFT JOIN sys_user_role ur
    ON ur.user_id = u.id
LEFT JOIN sys_role r
    ON r.id = ur.role_id
WHERE u.deleted = 0
  AND r.status = 1;
```

没有角色或没有启用角色的用户，其连接结果为：

```text
r.status = NULL
```

执行 `WHERE` 过滤时：

```sql
NULL = 1
```

结果为 `UNKNOWN`，该结果行会被过滤。这样就无法保留没有启用角色的用户，查询效果会接近 `INNER JOIN`。

