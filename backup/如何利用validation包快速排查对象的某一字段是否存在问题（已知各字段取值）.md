对于外部接收到的Java对象进行验证，可以使用 **Bean Validation (JSR 380)** 来实现。以下是完整的解决方案：

## 1. 添加依赖

```xml
<!-- Spring Boot 项目通常已包含 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- 或非Spring项目 -->
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>8.0.1.Final</version>
</dependency>
<dependency>
    <groupId>jakarta.validation</groupId>
    <artifactId>jakarta.validation-api</artifactId>
    <version>3.0.2</version>
</dependency>
```

## 2. 定义带验证注解的对象

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import java.util.List;

public class OrderRequest {
    
    @NotBlank(message = "订单号不能为空")
    @Size(min = 10, max = 50, message = "订单号长度必须在10-50之间")
    private String orderNo;
    
    @NotNull(message = "金额不能为null")
    @DecimalMin(value = "0.01", message = "金额必须大于0")
    private BigDecimal amount;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    // 嵌套对象 - 关键：使用 @Valid 触发级联验证
    @Valid
    @NotNull(message = "收货地址不能为null")
    private Address address;
    
    // List<嵌套对象> - 同样需要 @Valid
    @NotEmpty(message = "商品列表不能为空")
    @Valid
    private List<ProductItem> items;
    
    // getters and setters...
}

public class Address {
    
    @NotBlank(message = "省份不能为空")
    private String province;
    
    @NotBlank(message = "城市不能为空")
    private String city;
    
    @Pattern(regexp = "^\\d{6}$", message = "邮编格式不正确")
    private String zipCode;
    
    // getters and setters...
}

public class ProductItem {
    
    @NotNull(message = "商品ID不能为null")
    private Long productId;
    
    @Min(value = 1, message = "数量至少为1")
    private Integer quantity;
    
    @NotBlank(message = "商品名称不能为空")
    private String productName;
    
    // getters and setters...
}
```

## 3. 手动验证工具类（核心）

```java
import jakarta.validation.*;
import java.util.Set;

public class ValidationUtil {
    
    private static final Validator validator;
    
    static {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }
    
    /**
     * 验证对象，返回所有错误信息
     */
    public static <T> Set<ConstraintViolation<T>> validate(T object) {
        return validator.validate(object);
    }
    
    /**
     * 验证对象，抛出异常如果有错误
     */
    public static <T> void validateAndThrow(T object) {
        Set<ConstraintViolation<T>> violations = validator.validate(object);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
    
    /**
     * 获取友好的错误消息列表
     */
    public static <T> List<String> getErrorMessages(T object) {
        Set<ConstraintViolation<T>> violations = validator.validate(object);
        return violations.stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .collect(Collectors.toList());
    }
}
```

## 4. 使用示例

```java
public class ValidationDemo {
    
    public static void main(String[] args) {
        // 构造一个有问题的对象
        OrderRequest order = new OrderRequest();
        order.setOrderNo("123"); // 太短
        order.setAmount(new BigDecimal("-10")); // 负数
        order.setEmail("invalid-email"); // 格式错误
        
        Address address = new Address();
        address.setProvince(""); // 空字符串
        address.setCity(null); // null
        order.setAddress(address);
        
        ProductItem item1 = new ProductItem();
        item1.setProductId(null); // null
        item1.setQuantity(0); // 小于1
        order.setItems(Arrays.asList(item1));
        
        // 方式1：获取所有验证错误
        Set<ConstraintViolation<OrderRequest>> violations = ValidationUtil.validate(order);
        
        for (ConstraintViolation<OrderRequest> violation : violations) {
            System.out.println("字段: " + violation.getPropertyPath());
            System.out.println("错误: " + violation.getMessage());
            System.out.println("无效值: " + violation.getInvalidValue());
            System.out.println("---");
        }
        
        // 输出示例：
        // 字段: orderNo
        // 错误: 订单号长度必须在10-50之间
        // 无效值: 123
        // ---
        // 字段: amount
        // 错误: 金额必须大于0
        // 无效值: -10
        // ---
        // 字段: address.province
        // 错误: 省份不能为空
        // ---
        // 字段: items[0].productId
        // 错误: 商品ID不能为null
        // ---
        
        // 方式2：获取友好错误消息
        List<String> errors = ValidationUtil.getErrorMessages(order);
        errors.forEach(System.out::println);
        
        // 方式3：验证并抛异常
        try {
            ValidationUtil.validateAndThrow(order);
        } catch (ConstraintViolationException e) {
            e.getConstraintViolations().forEach(v -> 
                System.out.println(v.getPropertyPath() + ": " + v.getMessage())
            );
        }
    }
}
```

## 5. 关键要点

### ✅ 嵌套对象验证的关键
```java
@Valid  // 这个注解至关重要！没有它不会验证嵌套对象
private Address address;

@Valid  // List中的对象也需要
private List<ProductItem> items;
```

### ✅ 常用验证注解
| 注解 | 用途 |
|------|------|
| `@NotNull` | 不能为null |
| `@NotBlank` | 不能为null且去除空格后不为空 |
| `@NotEmpty` | 不能为null且集合/数组不为空 |
| `@Size(min, max)` | 字符串/集合长度范围 |
| `@Min/@Max` | 数值范围 |
| `@DecimalMin/@DecimalMax` | 小数范围 |
| `@Email` | 邮箱格式 |
| `@Pattern` | 正则表达式 |
| `@Valid` | **级联验证嵌套对象** |

### ✅ 自定义验证注解（可选）

```java
@Target({FIELD})
@Retention(RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface ValidPhone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<ValidPhone, String> {
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true; // null由@NotNull处理
        return value.matches("^1[3-9]\\d{9}$");
    }
}
```

## 6. 与Controller自动验证的区别

```java
// Controller中是自动的
@PostMapping("/order")
public Result create(@Valid @RequestBody OrderRequest order) { ... }

// 外部接收需要手动调用
OrderRequest order = receiveFromExternal();
Set<ConstraintViolation<OrderRequest>> violations = ValidationUtil.validate(order);
if (!violations.isEmpty()) {
    // 处理错误
}
```

这种方式可以**快速、全面地排查对象字段问题**，包括深层嵌套和集合中的对象，非常适合数据校验场景。