## Tomcat
- 基于 Java 实现的 HTTP Server + Servlet 容器
- 监听端口、接收并解析 HTTP 请求
- 按 Servlet 规范封装 Request / Response
- 调用 Servlet 处理请求
- 返回 HTTP 响应

## Servlet
- Java Web 的 HTTP 请求处理规范/API
- 定义 Request、Response、Servlet 等核心抽象

## Filter
Filter 是 Servlet 规范提供的过滤器，位于 Spring MVC 请求处理链的外围，可以在请求进入 Spring MVC 前、响应离开 Spring MVC 后进行处理

```java
@Component
public class LogFilter implements Filter {

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain
    ) throws IOException, ServletException {

        System.out.println("进入 LogFilter");

        chain.doFilter(request, response);

        System.out.println("离开 LogFilter");
    }
}
```

```java
@Configuration
public class FilterConfig {

    @Bean
    public FilterRegistrationBean<LogFilter> logFilterRegistration(
            LogFilter logFilter
    ) {
        FilterRegistrationBean<LogFilter> registration =
                new FilterRegistrationBean<>();

        // 使用 Spring 容器里的 LogFilter
        registration.setFilter(logFilter);

        // 只匹配 /api/*
        registration.addUrlPatterns("/api/*");

        // Filter 顺序，数字越小越靠外
        registration.setOrder(1);

        return registration;
    }
}
```

## HandlerInterceptor
HandlerInterceptor 是 Spring MVC 提供的拦截机制，用来在 Controller 执行前后插入统一逻辑

```java
@Component
public class LogInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler
    ) {
        System.out.println("1. Controller 之前");

        return true;
    }

    @Override
    public void postHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            ModelAndView modelAndView
    ) {
        System.out.println("3. Controller 执行完成");
    }

    @Override
    public void afterCompletion(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            Exception ex
    ) {
        System.out.println("4. 请求完成");
    }
}
```

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    private final LogInterceptor logInterceptor;

    public WebConfig(LogInterceptor logInterceptor) {
        this.logInterceptor = logInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        registry.addInterceptor(logInterceptor)
                .addPathPatterns("/api/**");
    }
}
```

## Validation

| 注解 | 含义 |
|---|---|
| `@NotNull` | 不能为 null |
| `@NotBlank` | 字符串不能是 null、空串或纯空白 |
| `@NotEmpty` | 字符串/集合等不能为空 |
| `@Min(18)` | 数值最小 18 |
| `@Max(100)` | 数值最大 100 |
| `@Size(min=2,max=10)` | 长度/大小范围 |
| `@Pattern(...)` | 正则校验 |
| `@Email` | 邮箱格式 |

```java
public class UserDTO {

    @NotBlank(message = "用户名不能为空")
    private String name;

    @Min(value = 18, message = "年龄不能小于18岁")
    private Integer age;
}
```

```java
@PostMapping("/user")
public void addUser(
    @Valid @RequestBody UserDTO dto
) {}

@PostMapping("/user")
public void create(
    @RequestParam
    @NotBlank
    String name
) {}
```

## 全局异常处理

@RestControllerAdvice 属于 Spring MVC 异常处理体系，Filter 抛出的异常，这边就没法捕获

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public Result<?> handleRuntimeException(RuntimeException e) {

        return Result.error(e.getMessage());
    }
}
```

## AOP

```
@Before          方法执行前
@After           方法结束后
@AfterReturning  方法正常返回后
@AfterThrowing   方法抛异常后
@Around          包围整个方法
```

```java
@Aspect
@Component
public class LogAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void before() {
        System.out.println("Service 方法准备执行");
    }
}
```

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface OperationLog {
    String value() default "";
}

@OperationLog("创建用户")
public void createUser(UserDTO dto) {
    ...
}

@Aspect
@Component
public class LogAspect {
    @Around("@annotation(operationLog)")
    public Object around(
            ProceedingJoinPoint joinPoint,
            OperationLog operationLog
    ) throws Throwable {

        System.out.println("操作：" + operationLog.value());

        Object result = joinPoint.proceed();

        return result;
    }
}
```