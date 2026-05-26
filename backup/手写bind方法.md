手写 `bind` 是 `call` / `apply` 的进阶版。它的核心不是“立即执行”，而是**制造一个带着固定上下文和预设参数的新函数**。下面给出面试级实现，并重点拆解最易踩坑的 `new` 行为。

---
### 📦 完整实现（覆盖 99% 面试场景）
```javascript
Function.prototype.myBind = function(context, ...presetArgs) {
  // 1. 类型守卫
  if (typeof this !== 'function') {
    throw new TypeError('myBind 只能被函数调用');
  }

  const targetFn = this;

  // 2. 返回的新函数
  function boundFn(...newArgs) {
    // 3. 判断是否通过 new 调用
    const isNew = this instanceof boundFn;
    // new 调用时忽略绑定 context，否则使用绑定的 context
    const ctx = isNew ? this : (context == null ? globalThis : Object(context));
    return targetFn.apply(ctx, [...presetArgs, ...newArgs]);
  }

  // 4. 原型链继承（支持 new 构造时访问原函数原型）
  boundFn.prototype = Object.create(targetFn.prototype ?? null);

  // 5. 同步 length 和 name（贴近原生行为）
  Object.defineProperty(boundFn, 'length', {
    value: Math.max(0, targetFn.length - presetArgs.length)
  });
  Object.defineProperty(boundFn, 'name', {
    value: `bound ${targetFn.name || 'anonymous'}`
  });

  return boundFn;
};
```

---
### 🔍 逐行拆解（重点看第 3、4 步）

| 代码片段 | 通俗解释 | 与 `call`/`apply` 的差异 |
|:---|:---|:---|
| `function(context, ...presetArgs)` | `bind` 接收上下文 + **任意数量的预设参数**（支持柯里化）。 | `call`/`apply` 收到参数后直接执行；`bind` 把参数“封存”起来，等将来调用时再合并。 |
| `function boundFn(...newArgs)` | **返回一个新函数**。新函数内部通过闭包记住了 `targetFn`、`context` 和 `presetArgs`。 | `call`/`apply` 没有返回新函数，而是直接返回执行结果。 |
| `const isNew = this instanceof boundFn` | **核心难点**：判断当前调用是否带了 `new`。如果 `new boundFn()`，`this` 就是新实例，`instanceof` 为 `true`。 | `call`/`apply` 不涉及 `new` 场景。 |
| `const ctx = isNew ? this : ...` | `new` 调用时，原生 `bind` 会**忽略绑定的 context**，让 `this` 指向新对象；否则使用绑定的 context。 | 保证 `bind` 返回的函数既能当普通函数用，也能当构造函数用。 |
| `targetFn.apply(ctx, [...presetArgs, ...newArgs])` | 合并预设参数和调用时传入的新参数，用正确的 `this` 执行原函数。 | 参数拼接顺序固定：`预设参数在前，新参数在后`。 |
| `boundFn.prototype = Object.create(targetFn.prototype ?? null)` | 让通过 `new boundFn()` 创建的实例，能访问原函数的原型方法。`?? null` 防止箭头函数等无原型时报错。 | `call`/`apply` 不需要处理原型链。 |
| `Object.defineProperty(... 'length' / 'name')` | 同步原生函数的元数据。`length` 减去已预设的参数个数；`name` 加上 `bound ` 前缀。 | 纯规范对齐，非必须但面试加分。 |

---
### 💡 为什么这么设计？3 个关键细节

1. **`new` 调用时为什么 `this instanceof boundFn` 为 `true`？**  
   JS 规范规定：`new` 操作符会创建一个空对象，并将其作为函数的 `this` 执行。此时 `this` 已经是 `boundFn` 的实例，所以 `instanceof` 判定为真。此时必须**放行**，让 `this` 指向新对象，否则构造函数行为会被破坏。

2. **为什么不用 `ctx.myBind = targetFn` 再执行？**  
   `bind` 返回的是**闭包**，不是临时挂载。闭包能安全隔离上下文，避免污染原对象，且支持多次调用、参数累加、延迟执行等场景。

3. **`presetArgs` 和 `newArgs` 的顺序为什么不能反？**  
   原生 `bind` 遵循“预设优先”原则：`fn.bind(obj, 1)(2)` 等价于 `fn.call(obj, 1, 2)`。如果顺序反了，会违背函数柯里化的数学直觉。

---
### 🌍 Java 视角对照

| 特性 | JavaScript `bind` | Java 等效思维 |
|:---|:---|:---|
| 返回值 | 返回一个**新函数**（闭包） | 返回一个 `Runnable` / `Consumer` / 自定义包装类实例 |
| `this` 绑定 | 创建时固化，运行时不可改（除非 `new`） | Java 的 `this` 永远指向当前对象实例，无法动态替换。需用**组合模式**或**上下文对象传参**模拟 |
| 预设参数 | 闭包捕获 `presetArgs`，后续调用自动拼接 | Java 可用 `PartialFunction`（Scala）或自定义 `FunctionWrapper` 缓存参数 |
| `new` 兼容 | `bind` 返回的函数仍可作构造函数，但 `this` 绑定失效 | Java 构造器不能被“绑定上下文”，反射调用 `Constructor.newInstance()` 总是创建新实例 |

**一句话类比：**  
> Java 的方法是“带固定工位的机器”，`bind` 相当于给这台机器**外包了一个带预设指令的遥控器**。遥控器出厂时已锁定操作台（`this`）和前半段工序（`presetArgs`），你随时按下启动键（调用），它会按预定流程跑完；但如果你非要把它当新机器生产线用（`new`），它会聪明地切换回新机器模式。

---
### ✅ 测试用例（覆盖核心场景）
```javascript
function greet(prefix, suffix) {
  return `${this.name}说：${prefix}${this.msg}${suffix}`;
}

const user = { name: 'Alice', msg: 'Hello' };

// 1. 普通绑定
const fn1 = greet.myBind(user, '【');
console.log(fn1('】')); // Alice说：【Hello】

// 2. 覆盖绑定无效（bind 是固化）
const fn2 = fn1.myBind({ name: 'Bob', msg: 'Hi' }, '《');
console.log(fn2('》')); // Alice说：《Hello》（context 仍为 user，preset 累加）

// 3. new 调用时 this 绑定失效，原型链正常
greet.prototype.shout = function() { return this.name + '!!!'; };
const Ctor = greet.myBind(null, '');
const inst = new Ctor('!!!');
console.log(inst.name); // undefined (新实例无 name)
console.log(inst.shout()); // undefined!!! (原型方法可用)

// 4. length 自动计算
console.log(greet.length); // 2
console.log(greet.myBind(user, 'x').length); // 1
```

---
### 📌 总结：`call` vs `apply` vs `bind`

| 方法 | 执行时机 | 参数形式 | 返回值 | 典型场景 |
|:---|:---|:---|:---|:---|
| `call` | 立即执行 | 独立参数 `a, b, c` | 函数执行结果 | 临时借用方法、明确传参 |
| `apply` | 立即执行 | 数组/类数组 `[a, b, c]` | 函数执行结果 | 参数来自数组、`Math.max` 等 |
| `bind` | **延迟执行** | 预设参数 `a, b` + 后续参数 | **新函数** | 事件回调、定时器、柯里化、固定上下文 |

`bind` 的本质是 **闭包 + 原型继承 + 规范对齐**。掌握它，就摸透了 JS 函数作为“一等公民”的全部核心能力。