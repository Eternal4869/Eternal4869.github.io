Spring 的事务管理基于 **AOP 代理机制** 和 **`ThreadLocal` 事务上下文** 实现。当某些场景破坏了代理调用链、异常传递规则或线程上下文时，`@Transactional` 就会“失效”。以下是开发中最常见的失效场景、底层原因及解决方案，按类别整理便于排查：

---
### 🔹 一、代理机制相关
| 场景 | 失效原因 | 解决方案 |
|------|----------|----------|
| **1. 同类内部方法自调用**<br>`this.methodB()` 调用带 `@Transactional` 的方法 | Spring 事务通过代理对象织入。`this` 指向目标对象而非代理，绕过了 AOP 切面，事务上下文不会传递。 | ① 注入自身 Bean 调用：`@Autowired private MyService self; self.methodB()`<br>② 开启 `exposeProxy=true` 后使用 `((MyService)AopContext.currentProxy()).methodB()`<br>③ 将方法拆分到不同 Bean |
| **2. 非 `public` 方法加 `@Transactional`** | Spring 默认只对 `public` 方法解析事务属性（`AbstractFallbackTransactionAttributeSource` 实现限制）。`protected`/`private`/包私有方法会被直接忽略。 | 改为 `public`。若必须非 public，需切换为 AspectJ 编译期织入（不推荐，通常改访问修饰符即可） |
| **3. 方法或类被 `final` 修饰** | CGLIB 代理基于继承，`final` 方法/类无法被重写，代理失效；JDK 动态代理要求必须实现接口。 | 移除 `final`，或让类实现接口并使用 JDK 代理（`@EnableTransactionManagement(proxyTargetClass = false)`） |

---
### 🔹 二、异常处理相关
| 场景 | 失效原因 | 解决方案 |
|------|----------|----------|
| **4. 异常被 `try-catch` 捕获且未抛出** | Spring 事务管理器依赖**未捕获异常**触发回滚。异常被吞掉后，框架认为方法正常执行，直接提交事务。 | ① catch 后手动标记回滚：<br>`TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();`<br>② 重新抛出运行时异常：`throw new RuntimeException(e);`（推荐） |
| **5. 抛出受检异常（Checked Exception）但未配置 `rollbackFor`** | 默认仅对 `RuntimeException` 和 `Error` 回滚。`IOException`、`SQLException` 等受检异常默认**提交**。 | 显式指定回滚规则：`@Transactional(rollbackFor = Exception.class)` 或指定具体异常类 |

---
### 🔹 三、线程与上下文相关
| 场景 | 失效原因 | 解决方案 |
|------|----------|----------|
| **6. 在 `@Async` 或新线程中操作数据库** | 事务上下文通过 `ThreadLocal` 绑定在当前线程。新线程无法继承父线程事务，各自独立或无事务。 | ① 避免在事务方法内开启新线程<br>② 若需异步，将事务边界移至异步方法内部<br>③ 复杂场景可使用 `TransactionSynchronizationManager` 手动传递（需谨慎） |
| **7. 手动 `new` 对象而非从容器获取** | `new MyService()` 创建的是普通 Java 对象，无 Spring 代理，事务切面根本不生效。 | 始终通过 `@Autowired`、构造函数注入或 `ApplicationContext` 获取 Bean |

---
### 🔹 四、配置与环境相关
| 场景 | 失效原因 | 解决方案 |
|------|----------|----------|
| **8. 数据库表引擎不支持事务** | 如 MySQL 使用 `MyISAM`，底层无事务能力，Spring 开启事务也无济于事。 | 使用支持事务的引擎（如 InnoDB），建表时指定 `ENGINE=InnoDB` |
| **9. 多数据源/多事务管理器未指定** | 项目存在多个 `PlatformTransactionManager`（如 JDBC、JPA、MyBatis），Spring 无法自动匹配，可能选错或报错。 | 通过 `@Transactional(transactionManager = "xxxTransactionManager")` 明确指定 |
| **10. 漏配事务依赖或自动配置被覆盖** | 未引入 `spring-boot-starter-jdbc`/`mybatis-spring-boot-starter` 等；或自定义配置类覆盖了 Boot 的默认事务自动装配。 | 检查依赖；Spring Boot 默认已开启事务，无需手动加 `@EnableTransactionManagement`，除非有冲突 |

---
### 🔹 五、传播行为与事务边界
| 场景 | 失效原因 | 解决方案 |
|------|----------|----------|
| **11. 传播行为配置不当** | 如 `Propagation.NOT_SUPPORTED`（挂起事务）、`Propagation.NEVER`（抛异常）、`Propagation.REQUIRES_NEW`（新建独立事务），导致预期回滚未发生。 | 根据业务选择传播级别。默认 `REQUIRED`（加入现有事务，无则新建）通常足够 |
| **12. `REQUIRES_NEW` 子事务回滚不影响父事务** | 新事务独立提交/回滚。子方法回滚后，父事务仍可能提交，造成数据不一致。 | 若需整体回滚，改用 `REQUIRED`；或父方法捕获异常后手动 `setRollbackOnly()` |

---
### 📌 核心排查 checklist
1. **是否走了代理？** 打断点看调用对象是 `$$EnhancerBySpringCGLIB$$` 还是原始类。
2. **异常是否透出？** 日志有无 `Transaction rolled back because it has been marked as rollback-only`。
3. **线程是否一致？** 打印 `Thread.currentThread().getName()` 确认未跨线程。
4. **引擎是否支持？** `SHOW CREATE TABLE xxx;` 检查 `ENGINE=InnoDB`。
5. **传播/回滚规则是否匹配？** 检查 `@Transactional` 参数是否符合业务预期。

> 💡 **经验总结**：Spring 事务不是“魔法”，而是 **AOP 代理 + 异常拦截 + ThreadLocal 上下文** 的组合。失效 90% 以上源于：
① 绕过代理调用 
② 异常被吞 
③ 跨线程 
④ 配置不匹配。
掌握底层机制后，配合日志与调试可快速定位。