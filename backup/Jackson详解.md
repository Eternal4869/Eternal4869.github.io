Jackson 是 Java 生态中最流行、性能最高的 JSON 处理库，也是 Spring Boot 默认的 JSON 处理器。鉴于你对 Java 编程细节、类型安全（尤其是集合和 Map）有较高要求，以下将从**基础用法、类型安全实践、常用注解、高级特性及 Spring Boot 最佳实践**五个维度系统梳理 Jackson 的核心用法。

---

### 1. 核心组件与基础用法
Jackson 的核心是 `ObjectMapper` 类。**注意：`ObjectMapper` 是线程安全的，在实际项目中应作为单例复用**，避免频繁创建导致性能下降和内存泄漏。

```java
import com.fasterxml.jackson.databind.ObjectMapper;

// 最佳实践：定义为静态常量或由 Spring 容器管理
private static final ObjectMapper mapper = new ObjectMapper();

public class User {
    private String name;
    private int age;
    // 省略 getter/setter 或使用了 Lombok @Data
}

// 1. 序列化：对象 -> JSON 字符串
User user = new User("Alice", 25);
String json = mapper.writeValueAsString(user); 
// 结果: {"name":"Alice","age":25}

// 2. 反序列化：JSON 字符串 -> 对象
User deserializedUser = mapper.readValue(json, User.class);
```

---

### 2. 集合与泛型的类型安全（重点）
由于 Java 的**泛型擦除**机制，如果直接将 JSON 反序列化为 `List.class` 或 `Map.class`，Jackson 只能将其解析为 `List<LinkedHashMap>`。当你尝试从中取出 `User` 对象时，会在运行时抛出 `ClassCastException`。

**正确做法：使用 `TypeReference` 保留泛型信息。**

```java
import com.fasterxml.jackson.core.type.TypeReference;

String jsonArray = "[{\"name\":\"Alice\",\"age\":25},{\"name\":\"Bob\",\"age\":30}]";
String jsonMap = "{\"user1\":{\"name\":\"Alice\",\"age\":25}}";

// ✅ 安全的 List 反序列化
List<User> userList = mapper.readValue(jsonArray, new TypeReference<List<User>>() {});

// ✅ 安全的 Map 反序列化 (Key 为 String, Value 为 User)
Map<String, User> userMap = mapper.readValue(jsonMap, new TypeReference<Map<String, User>>() {});

// 验证类型安全：取出的对象直接是 User 类型，无需强转，无运行时 ClassCastException 风险
User firstUser = userList.get(0); 
```

---

### 3. 常用核心注解
通过注解可以精细控制序列化/反序列化行为，无需修改全局配置。

| 注解 | 作用 | 示例 |
| :--- | :--- | :--- |
| `@JsonProperty` | 指定 JSON 字段名，解决 Java 驼峰与 JSON 下划线命名不一致的问题。 | `@JsonProperty("user_name") private String userName;` |
| `@JsonIgnore` | 序列化或反序列化时忽略该字段（如密码、内部状态）。 | `@JsonIgnore private String password;` |
| `@JsonFormat` | 格式化日期/时间或数字。 | `@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")` |
| `@JsonCreator` | 指定反序列化时使用的构造函数或工厂方法（常用于 `record` 或不可变类）。 | 配合 `@JsonProperty` 使用，见下方代码 |
| `@JsonValue` | 用于枚举类，指定序列化时输出的值（通常是枚举的 code 或 name）。 | `@JsonValue public String getCode() { return this.code; }` |

**不可变对象（如 Java Record）的反序列化示例：**
```java
public record OrderRecord(
    @JsonProperty("order_id") String orderId,
    @JsonProperty("total_amount") BigDecimal totalAmount
) {
    @JsonCreator // 告诉 Jackson 使用这个构造器进行反序列化
    public OrderRecord { } 
}
```

---

### 4. 高级特性与细节控制

#### 4.1 忽略未知属性（容错处理）
当前端或下游服务返回了 Java 类中不存在的字段时，Jackson 默认会抛出 `UnrecognizedPropertyException`。可以通过全局或局部配置忽略它们：
```java
// 全局配置（推荐在初始化 ObjectMapper 时设置）
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 局部配置（仅对当前类生效）
@JsonInclude(Include.NON_NULL) // 序列化时忽略 null 值
@JsonIgnoreProperties(ignoreUnknown = true) // 反序列化时忽略未知字段
public class User { ... }
```

#### 4.2 树模型 (Tree Model) 处理动态 JSON
当 JSON 结构不固定，或者你只需要提取其中某个深层字段而不想定义完整的 Java 类时，使用 `JsonNode` 树模型最高效。
```java
String complexJson = "{\"status\":200, \"data\":{\"user\":{\"id\":1, \"name\":\"Alice\"}}}";

JsonNode rootNode = mapper.readTree(complexJson);
// 安全地提取深层值，如果路径不存在会返回 MissingNode，不会抛 NPE
String name = rootNode.at("/data/user/name").asText(); 
int status = rootNode.get("status").asInt();
```

#### 4.3 自定义序列化/反序列化器
当默认行为无法满足需求时（例如特殊的金额格式、复杂的多态类型），可以实现自定义逻辑。
```java
// 自定义反序列化器示例：将特定字符串转为枚举
public class StatusDeserializer extends StdDeserializer<UserStatus> {
    public StatusDeserializer() { this(null); }
    public StatusDeserializer(Class<?> vc) { super(vc); }

    @Override
    public UserStatus deserialize(JsonParser jp, DeserializationContext ctxt) throws IOException {
        String code = jp.getText();
        return UserStatus.fromCode(code); // 你的自定义转换逻辑
    }
}
// 使用：@JsonDeserialize(using = StatusDeserializer.class)
```

---

### 5. Spring Boot 中的最佳实践

在 Spring Boot 项目中，**不要直接 `new ObjectMapper()`**，而应该复用 Spring 容器管理的 `ObjectMapper`，并通过官方推荐的方式进行定制，以免破坏 Spring Boot 的默认自动配置（如 Java 8 日期时间模块的支持）。

#### 方式一：通过 `application.yml` 配置（推荐用于简单需求）
```yaml
spring:
  jackson:
    default-property-inclusion: non_null # 序列化时忽略 null
    deserialization:
      fail-on-unknown-properties: false  # 忽略未知字段
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
```

#### 方式二：通过 `Jackson2ObjectMapperBuilderCustomizer`（推荐用于复杂/代码级配置）
这种方式可以保留 Spring Boot 的所有默认优化，同时叠加你的自定义规则。
```java
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import org.springframework.boot.autoconfigure.jackson.Jackson2ObjectMapperBuilderCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class JacksonConfig {

    @Bean
    public Jackson2ObjectMapperBuilderCustomizer customJackson() {
        return builder -> {
            // 全局命名策略：Java 驼峰自动映射为 JSON 下划线
            builder.propertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
            // 忽略未知属性
            builder.featuresToDisable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
            // 可以在此处注册自定义的 Module 或 Serializer/Deserializer
        };
    }
}
```

---

### 💡 核心避坑指南（针对技术细节）
1. **`Map` 的 Key 类型**：JSON 的 Key 只能是字符串。如果你尝试反序列化 `Map<Integer, User>`，Jackson 默认会把 Key 当作 `String` 处理，导致后续通过 `Integer` 查找时失败。解决方法是使用 `@JsonCreator` 自定义反序列化逻辑，或在反序列化后手动转换。
2. **`==` 与 `equals`**：Jackson 反序列化创建的是**新对象**。如果你依赖对象引用相等性（`==`）或将其放入 `HashSet`/`HashMap` 的 Key 中，务必确保你的实体类正确重写了 `equals()` 和 `hashCode()` 方法。
3. **性能优化**：如果涉及超大 JSON 的流式处理，避免使用 `readValue` 一次性加载到内存，应使用 `JsonParser` 进行流式读取（Streaming API）。