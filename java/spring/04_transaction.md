## 事务的定义

事务是一组操作组成的逻辑工作单元。这组操作要么全部成功并提交，要么在发生异常时全部失败并回滚，从而保证数据的一致性。

```java
beginTransaction();

try {
    createOrder();
    modOrder();
    commit();
} catch (RuntimeException | Error e) {
    rollback();
    throw e;
}
```
