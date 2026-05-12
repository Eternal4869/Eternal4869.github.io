在 Spring 框架中，**循环依赖（Circular Dependency）** 是指两个或多个 Bean 相互依赖，形成闭环。例如：`A → B → A`。Spring 对其有内置的解决机制，但**并非所有场景都能自动处理**。下面从原理、限制到实战解决方案进行系统梳理。

---
### 一、Spring 能解决的循环依赖场景
✅ **仅同时满足以下条件时，Spring 能自动解决：**
1. Bean 的作用域为 **Singleton**（单例）
2. 依赖注入方式为 **属性注入（Field）或 Setter 注入**（非构造器注入）
3. 未开启 `allowCircularReferences=false`（默认允许）

---
### 二、核心机制：三级缓存
Spring 通过 **三级缓存** 实现早期引用暴露，核心类为 `DefaultSingletonBeanRegistry`。

| 缓存级别 | 存储内容 | 作用 |
|:---|:---|:---|
| `singletonObjects`（一级） | 完整初始化的单例 Bean | 最终成品，直接对外提供 |
| `earlySingletonObjects`（二级） | 已实例化但未填充属性/初始化的 Bean（或代理对象） | 提前暴露引用，打破实例化死锁 |
| `singletonFactories`（三级） | `ObjectFactory<?>`，用于延迟创建早期引用 | 支持 AOP 代理的延迟生成 |

#### 🔄 解决流程（以 A → B → A 为例）
1. 创建 A，调用 `createBeanInstance()` 实例化后，**将 `ObjectFactory` 放入三级缓存**
2. 填充 A 的属性时发现依赖 B，触发 B 的创建
3. 创建 B，实例化后同样将 `ObjectFactory` 放入三级缓存
4. 填充 B 的属性时发现依赖 A，调用 `getSingleton("A", true)`：
   - 一级缓存无 → 二级缓存无 → 三级缓存找到 A 的 `ObjectFactory`
   - 调用 `factory.getObject()` 生成 **早期引用**（若 A 需 AOP，此时生成代理对象）
   - 将早期引用移入二级缓存，清除三级缓存中的 Factory
   - 返回早期引用给 B
5. B 完成属性填充与初始化，移入一级缓存，清理二三级缓存
6. 回到 A 的创建流程，B 已就绪，A 完成后续生命周期，移入一级缓存

---
### 三、为什么需要三级缓存？（AOP 代理的关键）
如果只用两级缓存，AOP 代理会带来问题：
- 早期引用可能是原始对象，但 AOP 要求在最终放入一级缓存时必须是代理对象。
- 三级缓存的 `ObjectFactory` 允许**延迟决定**返回原始对象还是代理对象。
- 当需要 AOP 时，`ObjectFactory.getObject()` 会调用 `createBean()` 生成代理，保证一级缓存中最终存放的是代理实例。

> 💡 Spring 源码入口：`AbstractBeanFactory.doGetBean()` → `DefaultSingletonBeanRegistry.getSingleton(beanName, allowEarlyReference)`

---
### 四、Spring **无法解决** 的循环依赖场景
| 场景 | 原因 | 异常 |
|:---|:---|:---|
| **构造器循环依赖** | 实例化阶段就需要依赖对象，无法提前暴露引用 | `BeanCurrentlyInCreationException` |
| **Prototype 作用域** | 每次请求都创建新实例，无法缓存早期引用 | 同上 |
| `@Configuration` 类之间的循环依赖 | 配置类生命周期特殊，早期引用机制不生效 | 启动失败 |
| 显式关闭循环依赖支持 | `ConfigurableApplicationContext.setAllowCircularReferences(false)` | 同上 |

---
### 五、实际项目中的解决方案（推荐顺序）

#### 🥇 1. 重构设计（根本解决）
循环依赖通常是**架构设计不合理**的信号。推荐做法：
- 提取公共服务类，打破双向依赖
- 使用事件机制（`ApplicationEventPublisher`）解耦
- 依赖倒置：让两个 Bean 依赖同一个接口，而非相互依赖

#### 🥈 2. 使用 `@Lazy` 延迟注入
在注入点加 `@Lazy`，Spring 会注入一个代理对象，首次调用方法时才真正解析依赖。
```java
@Service
public class A {
    private final B b;
    public A(@Lazy B b) { this.b = b; } // 构造器注入也适用
}
```
✅ 适用：构造器/字段注入、单例/原型  
⚠️ 注意：首次调用会有额外开销，且可能掩盖设计问题

#### 🥉 3. 改用 Setter/字段注入（仅限单例）
```java
@Service
public class A {
    @Autowired
    private B b; // Spring 可自动解决单例循环依赖
}
```
> 📌 官方建议：优先使用构造器注入，循环依赖场景可临时改用 Setter/字段注入。

#### 🔹 4. 使用 `ObjectProvider` 或 `ApplicationContext`
延迟获取 Bean，避免初始化期死锁：
```java
@Service
public class A {
    private final ObjectProvider<B> bProvider;
    public A(ObjectProvider<B> bProvider) { this.bProvider = bProvider; }
    
    public void doSomething() {
        B b = bProvider.getObject(); // 按需获取
    }
}
```

#### 🔹 5. 拆分模块或调整启动顺序
- 将循环依赖的 Bean 拆到不同 Spring 容器中（极少用）
- 使用 `@DependsOn` 明确初始化顺序（不解决循环，仅控制顺序）

---
### 六、Spring 6 / Spring Boot 3 的变化
- **核心机制未变**，三级缓存逻辑保持稳定
- **校验更严格**：部分边缘场景会直接报错而非静默处理
- **官方态度明确**：循环依赖属于“历史兼容特性”，**不推荐主动使用**，文档中多次强调应通过重构消除

---
### ✅ 总结与最佳实践
| 问题 | 建议 |
|:---|:---|
| 能否依赖 Spring 自动解决循环依赖？ | ❌ 仅限单例+Setter/字段注入，且是框架兜底机制 |
| 构造器循环依赖怎么办？ | ✅ 用 `@Lazy`、`ObjectProvider` 或重构 |
| 生产环境遇到循环依赖？ | 🛠 优先重构；紧急时用 `@Lazy` 过渡 |
| 如何预防？ | 模块化设计、依赖单向流动、CI 中集成 `spring-context` 依赖检查插件 |

> 📖 官方参考：[Spring Framework Documentation - Circular Dependencies](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html#beans-circular-dependencies)