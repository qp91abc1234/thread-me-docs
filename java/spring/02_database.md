## 数据库访问技术栈分层

- MyBatis-Plus（业务层调用）
  - 建立在 MyBatis 之上的增强框架
  - 提供大量通用 CRUD、条件构造、分页等能力，使基础数据库操作无需手写 SQL
  - 复杂或自定义查询仍可使用 MyBatis 的 SQL 能力
- MyBatis（业务层调用）
  - 基于 JDBC 的半自动 ORM（通常由开发者编写和控制 SQL）
  - 主要封装了繁琐的 JDBC 操作
  - 负责 SQL 参数绑定，数据库结果 ↔ Java 对象的映射
  - Mapper
    - Java 对象世界和关系数据库世界之间的“双向映射层”
    - 数据库查询结果 → Java 对象
    - Java 方法调用 + 参数 → SQL 语句 + SQL 参数
- Druid  
  - Druid 基于 JDBC Driver 创建的物理连接，对 Connection 进行池化、包装和管理
  - 核心目的就是连接池管理
  - 即调用，也实现/包装 JDBC 接口（ex.DataSource）
- JDBC
  - Java Database Connectivity（java 数据库连接）
  - 是 Java 定义的一套访问关系型数据库的规范，其中定义了很多接口；数据库厂商通过 JDBC Driver 来实现这些接口
  - DataSource
    - JDBC 定义的“连接来源”接口，具体实现通常持有数据库连接配置，核心作用是获取 Connection
  - Connection
    - JDBC 定义的“数据库连接”接口，具体实现由数据库驱动提供，代表 Java 程序与数据库服务端的一次连接，核心作用是创建 SQL 执行对象、管理事务等
- 数据库驱动
  - 数据库厂商按照 JDBC 规范提供的一套 Java 客户端实现，
  - 它向上实现 JDBC 接口供 Java 程序调用
  - 它向下通过数据库厂商的通信协议与数据库服务端通信
- 数据库软件
  - 负责数据的持久化存储、组织、查询、修改，以及事务、索引、权限等管理

## entity

- @TableName("user") 
  - 告知 mp 对应哪张表
  - 默认时：类名驼峰 → 表名下划线，并转小写
  - ex.User → user
- @TableId(type = IdType.AUTO)
  - 告知 mp 哪个属性是主键，用在 selectById 等接口
    - ex.主键是 uid 时，mp 调用 selectById 需要通过注解知道不是 by id，而是 by uid
  - 自增的作用是告诉 mp 插入时不要给主键赋值，mp 默认是赋值，在插入时会给主键赋值

## mapper

MP 提供的数据库映射层
- Java → SQL
- 数据库结果 → Java 对象

MP 的 BaseMapper 会自动生成单表常见的 crud，自定义 XML/注解 SQL用于补充自定义 SQL

```
src/main/java
└── com.example.demo
    └── mapper
        └── UserMapper.java

src/main/resources
└── mapper
    └── UserMapper.xml
```

```java
@Mapper // 告诉 MyBatis 这是一个 Mapper 接口，Spring 会扫描并创建代理实现
public interface UserMapper extends BaseMapper<User> {
    /*
     * BaseMapper 已经提供了常用的单表 CRUD，例如：
     * insert、deleteById、updateById、selectById、selectList。
     * 简单 CRUD 不需要在这里重复声明。
     */

    /**
     * 自定义 SQL 示例：根据姓名关键字进行模糊查询。
     *
     * <p>当 BaseMapper 和条件构造器不能清晰表达查询时，可以在 Mapper 中写 SQL。
     * 简短 SQL 可以使用注解；复杂 SQL 通常放在 XML 文件中。</p>
     */
    @Select("SELECT id, name, age FROM `user` "
            + "WHERE name LIKE CONCAT('%', #{keyword}, '%') ORDER BY id")
    List<User> selectByNameLike(@Param("keyword") String keyword);

    List<UserVO> selectUserStatistics(Long deptId);
}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.demo.mapper.UserMapper">

    <select id="selectUserStatistics"
            resultType="com.example.demo.vo.UserVO">

        SELECT ...
        FROM user u
        WHERE u.dept_id = #{deptId}

    </select>

</mapper>
```

## service

IService<User>
→ 定义标准
→ mp 在 Service 层提供的一套通用 CRUD 接口，
  定义 getById、save、list、updateById、removeById 等能力。

ServiceImpl<UserMapper, User>
→ 提供标准实现
→ mp 提供的 IService 通用实现类，
  内部结合 UserMapper 完成具体的数据库 CRUD 操作。

UserService extends IService<User>
→ 扩展标准
→ 项目自己的 Service 接口。
→ 继承 IService<User> 的通用 CRUD 能力，
  同时可以定义 User 业务特有的方法。

UserServiceImpl
    extends ServiceImpl<UserMapper, User>
    implements UserService
→ 继承标准实现 + 实现扩展
→ 项目自己的 Service 实现类。
→ 继承 ServiceImpl<UserMapper, User> 已经实现好的通用 CRUD，
  同时负责实现 UserService 中额外定义的业务方法。