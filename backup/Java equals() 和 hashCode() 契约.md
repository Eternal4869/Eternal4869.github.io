在Java中，`equals()` 和 `hashCode()` 是 `java.lang.Object` 类中的两个核心方法。它们不仅是判断对象相等性的基础，更是所有**基于哈希的集合类**（如 `HashMap`, `HashSet`, `Hashtable`）正常工作的基石。

所谓“契约（Contract）”，就是**当你决定重写其中一个方法时，必须同时重写另一个方法，并且它们之间必须严格遵守一套逻辑规则**。如果不遵守，程序虽然不会报错，但会在哈希集合中产生极其诡异的Bug。

---

### 一、 核心契约：三大黄金法则

这是面试必考、开发必知的核心规则：

1. **一致性法则（最重要）**：
   如果两个对象通过 `equals()` 比较返回 `true`（逻辑相等），那么它们的 `hashCode()` 返回值**必须相等**。
2. **冲突允许法则**：
   如果两个对象通过 `equals()` 比较返回 `false`（逻辑不相等），它们的 `hashCode()` **可以相等，也可以不相等**。（但尽量不相等，以减少哈希冲突，提高性能）。
3. **逆否命题法则（哈希表快速排除的依据）**：
   如果两个对象的 `hashCode()` **不相等**，那么它们通过 `equals()` 比较**一定返回 `false`**。

*(注：如果两个对象的 `hashCode()` 相等，它们 `equals()` **不一定**相等，这种情况称为“哈希冲突”。)*

---

### 二、 通俗比喻：小区找人与哈希表原理

为了更好理解，我们把对象放入 `HashMap` 想象成**在小区里找人**：

* **`hashCode()` 相当于“楼栋号”**：哈希表通过哈希算法快速计算出对象应该放在哪个“桶（Bucket/楼栋）”里。
* **`equals()` 相当于“门牌号+人脸比对”**：如果楼栋号相同（发生哈希冲突），就需要进到这栋楼里，挨个房间用 `equals()` 确认到底是不是你要找的那个人。

**结合比喻理解契约：**
* **法则1**：如果两个人是**同一个人**（`equals` 为 true），那他们**肯定住在同一栋楼**（`hashCode` 相等）。
* **法则3**：如果两个人**不住在同一栋楼**（`hashCode` 不等），那他们**绝对不是同一个人**（`equals` 为 false）。*（这就是为什么哈希表查找速度极快，先算楼栋号，楼栋号不对直接排除，根本不需要去比对人脸）。*
* **哈希冲突**：如果两个人**住在同一栋楼**（`hashCode` 相等），他们**不一定是同一个人**（`equals` 可能为 false，可能只是邻居）。

---

### 三、 违反契约的灾难后果（代码演示）

假设我们有一个 `Student` 类，我们**只重写了 `equals()`，却忘记重写 `hashCode()`**。

```java
import java.util.HashSet;
import java.util.Objects;

class Student {
    private String name;
    private int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // 1. 我们重写了 equals，认为名字和年龄相同就是同一个学生
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Student student = (Student) o;
        return age == student.age && Objects.equals(name, student.name);
    }

    // ❌ 致命错误：没有重写 hashCode()！
    // 此时 hashCode 默认使用 Object 类的实现（基于内存地址计算）
}

public class ContractViolationTest {
    public static void main(String[] args) {
        Student s1 = new Student("张三", 20);
        Student s2 = new Student("张三", 20);

        // 验证契约：
        System.out.println("s1 equals s2? " + s1.equals(s2)); // true (逻辑上相等)
        System.out.println("s1 hashCode: " + s1.hashCode());  // 例如：1234567
        System.out.println("s2 hashCode: " + s2.hashCode());  // 例如：7654321 (内存地址不同)
        
        // ❌ 违反了契约第一条：equals为true，但hashCode不相等！

        // 灾难发生：将它们放入 HashSet
        HashSet<Student> set = new HashSet<>();
        set.add(s1);
        set.add(s2);

        // 预期：集合里只有1个张三。
        // 实际：集合里有2个张三！因为 HashSet 先比较 hashCode，发现不同，
        // 直接认为它们是不同的对象，连 equals 都没调用就放进去了。
        System.out.println("Set size: " + set.size()); // 输出 2 (Bug!)
    }
}
```

**在 `HashMap` 中同理**：如果你用这个 `Student` 作为 Key 存入 `map.put(s1, 100)`，当你用 `map.get(s2)` 去取时，会因为 `s2` 的 `hashCode` 和 `s1` 不同，导致 `HashMap` 去错误的“楼栋”里找，最终返回 `null`。

---

### 四、 如何正确重写这两个方法？

在实际开发中，重写这两个方法有几种标准姿势：

#### 1. 传统手写（Java 7 之前）
需要引入质数（通常是 31）进行计算，代码繁琐且容易出错。

#### 2. 现代标准写法（推荐：使用 `java.util.Objects`）
Java 7 引入了 `Objects` 工具类，极大地简化了重写过程。

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    Student student = (Student) o;
    // 使用 Objects.equals 避免空指针，自动处理基本类型和引用类型
    return age == student.age && Objects.equals(name, student.name);
}

@Override
public int hashCode() {
    // 将参与 equals 比较的所有字段传入 hash 方法
    return Objects.hash(name, age);
}
```

#### 3. 终极偷懒写法（使用 Lombok）
在实际企业开发中，通常使用 Lombok 注解自动生成，既简洁又不容易出错。

```java
import lombok.EqualsAndHashCode;
import lombok.AllArgsConstructor;

@AllArgsConstructor
@EqualsAndHashCode // 加上这个注解，Lombok 会自动生成基于所有非静态字段的 equals 和 hashCode
public class Student {
    private String name;
    private int age;
}
```
*(注：如果只想用部分字段，可以使用 `@EqualsAndHashCode(of = {"name"})` 或 `@EqualsAndHashCode(exclude = {"age"})`)*

---

### 五、 总结与避坑指南

1. **铁律**：**重写 `equals()` 时，必须同时重写 `hashCode()`**。反之亦然（虽然很少见只重写 hashCode 的）。
2. **参与计算的字段必须一致**：在 `equals()` 中用来判断相等的字段，**必须全部**参与到 `hashCode()` 的计算中。如果 `equals` 用到了 `name` 和 `age`，`hashCode` 里绝不能漏掉 `age`。
3. **不要使用易变字段**：尽量**不要**将对象中会被修改的字段（如 `Mutable` 对象）作为 `hashCode` 和 `equals` 的计算依据。如果一个对象作为 `HashMap` 的 Key，存入后你修改了它的属性导致 `hashCode` 变了，这个对象在 `HashMap` 中就**永远丢失（找不到）** 了。
4. **性能考量**：`hashCode()` 的计算应该尽量高效，且计算结果应尽量散列（均匀分布），以减少哈希冲突。`String` 类的 `hashCode` 会缓存计算结果，就是为了提高性能。