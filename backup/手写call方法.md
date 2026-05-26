手写 `call` 是前端面试的经典题目。它的核心思想非常直观：**借壳执行**。

---
### 📦 完整实现（生产可用版）
```javascript
Function.prototype.myCall = function(context, ...args) {
  // 1. 类型守卫：只有函数才能调用 call
  if (typeof this !== 'function') {
    throw new TypeError('myCall 只能被函数调用');
  }

  // 2. 处理 context：null/undefined 时默认指向全局，原始值包装为对象
  const ctx = context == null ? globalThis : Object(context);

  // 3. 生成唯一标识，避免覆盖目标对象原有属性
  const fnKey = Symbol('myCallFn');

  // 4. 把当前函数“临时挂载”到目标对象上
  ctx[fnKey] = this;

  // 5. 执行函数，此时函数内部的 this 自然指向 ctx
  const result = ctx[fnKey](...args);

  // 6. 清理临时属性，不留痕迹
  delete ctx[fnKey];

  // 7. 返回原函数的执行结果
  return result;
};
```

---
### 🔍 逐行通俗讲解（附 Java 视角对照）

| 代码片段 | 通俗解释 | Java 视角对照 |
|:---|:---|:---|
| `Function.prototype.myCall = function(...)` | 给所有函数装上这个方法。JS 中函数也是对象，挂载到原型上就能全局使用。 | Java 没有“原型链”，相当于给某个基类或工具类写了个静态扩展方法（如 `MethodUtil.invoke(obj, args)`）。 |
| `if (typeof this !== 'function')` | 防止误用：`123.myCall()` 或 `null.myCall()` 会直接报错。 | Java 编译期就限制了调用者必须是对象实例，JS 运行时才检查，所以需手动防御。 |
| `context == null ? globalThis : Object(context)` | 处理边界：传 `null` 或 `undefined` 时指向全局；传数字/字符串等原始值时，用 `Object()` 包装成对象，否则挂载属性会失败。 | Java 的 `this` 永远是对象引用，不存在原始值。JS 的 `this` 可以是任意类型，但只有对象能“挂载方法”。 |
| `const fnKey = Symbol('myCallFn')` | 生成一个绝对唯一的键。不用字符串 `'fn'` 是因为可能和目标对象原有属性名冲突，导致数据被覆盖。 | Java 反射中动态注入方法也会用唯一标识或临时代理类，避免污染原对象结构。 |
| `ctx[fnKey] = this` | **核心步骤**：把调用 `myCall` 的那个函数，临时变成 `ctx` 的一个属性。 | 相当于把某个类的静态方法“动态绑定”到某个实例上，让它的 `this` 指向该实例。 |
| `const result = ctx[fnKey](...args)` | 执行函数。因为函数现在是 `ctx` 的属性，所以执行时内部的 `this` 自动指向 `ctx`，参数也完整传入。 | 类似 `instance.method(args)`，但 Java 的 `this` 是编译期写死的，JS 是运行时决定的。 |
| `delete ctx[fnKey]` | 用完就删，恢复对象原貌。不删会导致对象多出一个隐藏属性。 | Java 没有这种机制，内存由 GC 自动管理，无需手动“卸载方法”。 |
| `return result` | 把原函数的返回值交还出去，保持和原生 `call` 行为一致。 | 同 Java 方法调用的返回值传递。 |

---
### 💡 为什么这么设计？3 个关键细节

1. **为什么不用 `ctx.fn = this`？**  
   如果目标对象本身就有 `fn` 属性，直接赋值会覆盖原有数据。`Symbol` 是 ES6 引入的唯一值类型，能 100% 避免命名冲突。

2. **为什么用 `context == null` 而不是 `!context`？**  
   `0`、`''`、`false` 在 JS 中是合法的上下文对象，`!context` 会把它们误判为“空”而替换成全局对象。`== null` 只拦截 `null` 和 `undefined`，更符合规范。

3. **严格模式下的 `null` 处理**  
   原生 `call` 在严格模式下传 `null`，`this` 会保持为 `null`（不会指向全局）。上述代码为兼容老代码做了默认全局处理。若需 100% 贴合现代规范，可改为：
   ```javascript
   const ctx = context == null ? (strictMode ? context : globalThis) : Object(context);
   ```
   面试中写出 `context == null ? globalThis : Object(context)` 已完全够用。

---
### 🌍 Java vs JS：`this` 的本质差异

| 特性 | Java | JavaScript |
|:---|:---|:---|
| `this` 绑定时机 | **编译期静态绑定**：`this` 永远指向声明该方法的类的实例 | **运行时动态绑定**：`this` 由“谁调用”决定，与定义位置无关 |
| 方法归属 | 方法属于类，不能脱离类存在 | 函数是一等公民，可独立存在、传递、挂载 |
| `call` 的作用 | 不需要。Java 的调用方式固定：`obj.method()` | 提供“动态切换调用者”的能力，实现函数复用和上下文劫持 |

**一句话总结 JS 的 `call`：**  
> Java 的 `this` 是“认祖归宗”（写死在类里），JS 的 `this` 是“谁调用就认谁”。`call` 就是给函数发了一个“临时身份证”，让它假装是某个对象的方法去执行，干完活再把身份证收走。

---
### ✅ 测试用例（验证实现）
```javascript
function greet(age) {
  return `${this.name} 今年 ${age} 岁`;
}

const person = { name: '张三' };

console.log(greet.myCall(person, 25)); // "张三 今年 25 岁"
console.log(greet.myCall(null, 30));   // 指向全局（非严格模式）
console.log(greet.myCall(0, 18));      // 原始值 0 被包装为 Number 对象
```