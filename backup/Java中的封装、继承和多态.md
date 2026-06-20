在Java及所有面向对象编程语言中，**封装、继承、多态**被称为面向对象编程（OOP）的三大基本特性。它们共同构成了Java构建复杂、可维护、可扩展软件系统的基础。

---

### 一、 封装 (Encapsulation)

#### 1. 什么是封装？
封装是指**将对象的属性（状态）和行为（方法）结合成一个独立的整体，并尽可能隐藏对象的内部实现细节**，仅对外暴露必要的公共接口来进行访问和修改。
通俗地说：就像用胶囊把药粉包起来，你不需要知道药粉是怎么混合的，只需要吞下胶囊（调用接口）就能治病。

#### 2. 为什么要封装？（优点）
*   **安全性**：防止外部代码随意修改对象的核心数据，避免数据被破坏。
*   **易维护性**：内部实现细节改变时，只要对外接口不变，就不会影响外部调用者（高内聚，低耦合）。
*   **可控性**：可以在 setter 方法中加入逻辑校验，控制数据的合法性。

#### 3. 如何实现？
在Java中，主要通过**访问权限修饰符**（`private`, `default`, `protected`, `public`）来实现。通常将属性设为 `private`，然后提供 `public` 的 `getter` 和 `setter` 方法。

#### 4. 代码示例：银行账户
```java
public class BankAccount {
    // 1. 将属性私有化，外部无法直接访问和修改
    private String accountNo;
    private double balance;

    public BankAccount(String accountNo, double initialBalance) {
        this.accountNo = accountNo;
        // 利用封装进行数据校验
        if (initialBalance >= 0) {
            this.balance = initialBalance;
        } else {
            throw new IllegalArgumentException("初始余额不能为负数");
        }
    }

    // 2. 提供公共的存款方法（隐藏了余额增加的内部细节）
    public void deposit(double amount) {
        if (amount > 0) {
            this.balance += amount;
            System.out.println("存款成功，当前余额: " + balance);
        } else {
            System.out.println("存款金额必须大于0");
        }
    }

    // 3. 提供公共的取款方法（加入了余额是否充足的校验逻辑）
    public void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            this.balance -= amount;
            System.out.println("取款成功，当前余额: " + balance);
        } else {
            System.out.println("取款失败：金额无效或余额不足");
        }
    }

    // 4. 余额通常只允许查看，不允许外部随意修改，所以只提供 getter
    public double getBalance() {
        return balance;
    }
}
```
**测试：**
```java
BankAccount myAccount = new BankAccount("62220000", 1000);
// myAccount.balance = -500; // 编译报错！外部无法直接修改私有属性
myAccount.withdraw(2000); // 输出：取款失败：金额无效或余额不足（内部逻辑保护了数据）
```

---

### 二、 继承 (Inheritance)

#### 1. 什么是继承？
继承是指**让一个类（子类）获得另一个类（父类）的属性和方法**。它体现了现实世界中“is-a”（是一个）的关系。例如：“狗”是一个“动物”。

#### 2. 为什么要继承？（优点）
*   **代码复用**：子类可以直接使用父类已有的代码，减少重复编写。
*   **建立类之间的层次关系**：为后面的“多态”打下基础。

#### 3. 如何实现？
在Java中使用 `extends` 关键字。
*注意：Java**只支持单继承**（一个子类只能有一个直接父类），但支持多层继承。所有类最终都继承自 `java.lang.Object` 类。*

#### 4. 代码示例：动物体系
```java
// 父类
class Animal {
    protected String name; // 使用 protected，允许子类访问

    public Animal(String name) {
        this.name = name;
    }

    public void eat() {
        System.out.println(name + " 正在吃东西...");
    }
    
    public void sleep() {
        System.out.println(name + " 正在睡觉...");
    }
}

// 子类继承父类
class Dog extends Animal {
    
    // 子类特有的属性
    private String breed; 

    public Dog(String name, String breed) {
        super(name); // 调用父类的构造方法
        this.breed = breed;
    }

    // 子类特有的方法
    public void bark() {
        System.out.println(name + " 汪汪叫！");
    }
}

class Cat extends Animal {
    public Cat(String name) {
        super(name);
    }
    
    public void catchMouse() {
        System.out.println(name + " 正在抓老鼠...");
    }
}
```
**测试：**
```java
Dog dog = new Dog("旺财", "中华田园犬");
dog.eat();       // 继承自父类的方法，输出：旺财 正在吃东西...
dog.sleep();     // 继承自父类的方法
dog.bark();      // 子类自己的方法

Cat cat = new Cat("汤姆");
cat.catchMouse(); // 子类自己的方法
```

---

### 三、 多态 (Polymorphism)

#### 1. 什么是多态？
多态是指**同一个行为（方法调用），具有多个不同表现形式或形态**。
通俗地说：同样是“叫”这个动作，狗叫是“汪汪”，猫叫是“喵喵”。在代码层面表现为：**父类的引用变量，指向子类的对象，调用同一个方法时，表现出不同的行为。**

#### 2. 多态存在的三个前提条件：
1.  **继承**（或实现接口）。
2.  **方法重写**（子类重写父类的方法）。
3.  **父类引用指向子类对象**（向上转型）。

#### 3. 为什么要多态？（优点）
*   **消除类型之间的耦合关系**：提高代码的扩展性。
*   **统一接口调用**：不需要知道对象的具体类型，就能调用其方法。

#### 4. 代码示例：多态的体现与类型转换
我们继续完善上面的 `Animal` 体系，让子类**重写**父类的 `eat()` 方法。

```java
// 修改父类和子类，加入方法重写
class Animal {
    public void eat() {
        System.out.println("动物在吃.generic食物");
    }
}

class Dog extends Animal {
    @Override // 建议加上注解，检查是否重写成功
    public void eat() {
        System.out.println("狗在啃骨头");
    }
    public void guardHouse() {
        System.out.println("狗在看家护院");
    }
}

class Cat extends Animal {
    @Override
    public void eat() {
        System.out.println("猫在吃鱼");
    }
}
```

**多态的核心演示：**
```java
public class PolymorphismTest {
    public static void main(String[] args) {
        // 1. 向上转型：父类引用指向子类对象
        Animal myDog = new Dog(); 
        Animal myCat = new Cat();

        // 2. 多态的体现：调用同一个 eat() 方法，执行的是子类重写后的逻辑
        myDog.eat(); // 输出：狗在啃骨头
        myCat.eat(); // 输出：猫在吃鱼
        
        // 3. 多态的局限性：
        // 父类引用只能调用父类中声明过的方法。
        // myDog.guardHouse(); // 编译报错！Animal类中没有guardHouse方法
        
        // 4. 向下转型：如果确实需要调用子类特有的方法，需要进行强制类型转换
        // 为了安全，转换前必须使用 instanceof 关键字进行判断
        if (myDog instanceof Dog) {
            Dog realDog = (Dog) myDog; // 向下转型
            realDog.guardHouse();      // 输出：狗在看家护院
        }
    }
    
    // 5. 多态在实际开发中的最大威力：作为方法参数，提高扩展性
    public static void feedAnimal(Animal animal) {
        System.out.print("主人开始喂食: ");
        animal.eat(); // 传入什么对象，就执行什么对象的 eat() 方法
    }
}
```
**扩展性体现：**
```java
// 假设未来我们又增加了一个 Pig 类
class Pig extends Animal {
    @Override
    public void eat() { System.out.println("猪在吃饲料"); }
}

// 我们不需要修改 feedAnimal 方法的代码，直接传入新对象即可
PolymorphismTest.feedAnimal(new Dog()); // 主人开始喂食: 狗在啃骨头
PolymorphismTest.feedAnimal(new Pig()); // 主人开始喂食: 猪在吃饲料
```

---

### 四、 总结与串联

这三大特性不是孤立的，而是相辅相成的：
1.  **封装**是基础，它保证了数据的安全和模块的独立，让类成为一个可靠的“黑盒”。
2.  **继承**是纽带，它建立了类与类之间的层级关系，实现了代码的复用。
3.  **多态**是升华，它基于封装和继承，利用“父类引用指向子类对象”和“动态绑定（运行时决定调用哪个方法）”的机制，让系统具备极强的**灵活性和可扩展性**（符合面向对象设计原则中的“开闭原则”：对扩展开放，对修改封闭）。