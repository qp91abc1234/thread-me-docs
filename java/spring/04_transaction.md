## 事务的定义

事务是一组操作组成的逻辑工作单元。这组操作要么全部成功并提交，要么在发生异常时全部失败并回滚，从而保证数据的一致性。

```java
@Service
public class OrderService {

    @Transactional
    public void handleOrder() {
        createOrder();
        modOrder();
    }
}
```

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

## @Transactional 与代理对象

Spring 会为需要事务管理的 Bean 创建代理对象，之后其他 Bean 从 Spring 容器中获取或注入该 Bean 时，拿到的通常也是这个代理对象。

```text
其他 Bean 注入 OrderService
        ↓
获得 OrderService 代理对象
        ↓
代理对象开启事务
        ↓
调用原始 OrderService 对象的方法
        ↓
方法体中的 this 指向原始对象
```

```java
public class OrderServiceProxy implements OrderService {

    // 真正执行业务逻辑的对象
    private final OrderService target;

    public OrderServiceProxy(OrderService target) {
        this.target = target;
    }

    @Override
    public void handleOrder() {
        beginTransaction();

        try {
            // 调用真正的业务方法
            target.handleOrder();
            commit();
        } catch (RuntimeException | Error e) {
            rollback();
            throw e;
        }
    }
}
```

## @Transactional 生效条件

- 所在对象必须由 Spring 管理，例如使用 `@Service`。
- 通常标注在 `public` 方法上。
- 方法需要通过 Spring 创建的代理对象被外部调用。
- 操作的数据库需要支持事务。
- 多个操作需要使用同一个事务管理器和数据源。
- 新线程和 `@Async` 方法不会自动加入当前事务。

## 事务的回滚规则

1. 默认情况下，`RuntimeException` 和 `Error` 会触发回滚。
2. 受检异常默认不回滚，可以通过 `rollbackFor` 配置。
3. 异常必须传播到事务代理对象；在方法内部捕获且不再抛出，事务通常会提交。

### 通过 rollbackFor 配置受检异常回滚

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() throws IOException {
    saveOrder();
    throw new IOException("文件写入失败");
}
```

`IOException` 是受检异常，默认不会触发回滚。配置 `rollbackFor = Exception.class` 后，它也会触发回滚。

### 捕获异常后必须继续抛出

```java
@Transactional
public void createOrder() {
    try {
        saveOrder();
        decreaseStock();
    } catch (RuntimeException e) {
        log.error("创建订单失败", e);
        throw e;
    }
}
```

如果 `catch` 中只记录日志而不继续抛出异常，代理对象会认为方法正常结束，因此通常会提交事务。

## 事务传播

事务传播解决的是：一个事务方法调用另一个事务方法时，被调用方法应该加入当前事务，还是创建一个新事务。

`propagation` 通常配置在被调用方法上，约束的是该方法被调用时如何处理调用方已经存在的事务。

`REQUIRED` 是默认传播方式：存在事务就加入现有事务，不存在事务就创建新事务。

`REQUIRES_NEW` 会暂停当前事务，并为被调用方法创建一个独立的新事务。

```java
@Service
public class OrderService {

    private final StockService stockService;

    @Transactional
    public void createOrder() {
        saveOrder();
        stockService.decreaseStock();
    }
}
```

```java
@Service
public class StockService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void decreaseStock() {
        updateStock();
    }
}
```

调用 `decreaseStock()` 时已经存在外层事务，因此它会加入 `createOrder()` 的事务。两个方法中的数据库操作一起提交或一起回滚。

```text
存在事务 → 加入现有事务
不存在事务 → 创建新事务
```

### 内层异常与 rollbackOnly

`REQUIRED` 的内外层只有一个物理事务；内层异常经过内层代理时，会把这个共享事务标记为 `rollbackOnly`。外层即使捕获异常，也不能再把它恢复为可提交状态。

```java
@Transactional
public void createOrder() {
    saveOrder();

    try {
        stockService.decreaseStock();
    } catch (RuntimeException e) {
        log.error("扣减库存失败", e);
    }
}
```

```java
@Transactional
public void decreaseStock() {
    updateStock();
    throw new RuntimeException("库存更新失败");
}
```

执行过程：

```text
外层开启事务 A
    ↓
内层加入事务 A
    ↓
内层抛出 RuntimeException
    ↓
内层代理将事务 A 标记为 rollbackOnly
    ↓
外层捕获异常并尝试提交
    ↓
事务 A 只能回滚，并可能抛出 UnexpectedRollbackException
```
