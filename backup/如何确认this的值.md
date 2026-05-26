JavaScript 中 `this` 的指向**只与调用方式有关，与定义位置无关**。严格模式与非严格模式的核心差异在于：**当没有明确的调用对象时，严格模式拒绝“隐式回退到全局对象”，而是直接返回 `undefined`。**

以下是不同场景下的详细对比（以浏览器环境为例）：

### 📊 核心行为对比表
| 调用场景 | 非严格模式 `this` | 严格模式 `this` | 是否受模式影响 |
|:---|:---|:---|:---|
| **全局作用域**（脚本顶层） | `window` | `undefined` | ✅ |
| **普通函数调用** `fn()` | `window` | `undefined` | ✅ |
| **对象方法调用** `obj.fn()` | `obj` | `obj` | ❌ |
| **构造函数** `new Fn()` | 新实例 | 新实例 | ❌ |
| **`call/apply/bind(null)`** | `window` | 保持 `null` | ✅ **重要差异** |
| **箭头函数** `() => {}` | 继承外层作用域的 `this` | 继承外层作用域的 `this` | ❌（但外层受模式影响） |
| **DOM 事件回调** | 触发事件的 DOM 元素 | 触发事件的 DOM 元素 | ❌（浏览器强制绑定） |
| **定时器/异步回调** `setTimeout(fn)` | `window` | `undefined` | ✅ |
| **ES6 Class 方法内部** | 实例对象 | 实例对象 | ❌（类体默认严格模式） |

---

### 🔍 关键场景详解

#### 1. 普通函数调用（最常见陷阱）
```js
function foo() {
  console.log(this);
}
// 非严格模式
foo(); // window
// 严格模式
"use strict";
foo(); // undefined
```
**为什么重要？** 非严格模式下，漏写 `this` 或误用变量会默默污染全局；严格模式直接抛出 `TypeError: Cannot read properties of undefined`，强制你显式绑定或改用箭头函数。

#### 2. `call / apply / bind` 传入 `null` 或 `undefined`
```js
function show() { console.log(this); }

// 非严格模式
show.call(null);       // window
show.apply(undefined); // window

// 严格模式
"use strict";
show.call(null);       // null
show.apply(undefined); // undefined
```
**规范意义**：严格模式尊重开发者传入的原始值，不再做“全局对象兜底”的隐式转换。

#### 3. 对象方法 & 构造函数（规则一致）
```js
const obj = {
  name: "test",
  say() { console.log(this.name); }
};
obj.say(); // 无论严格/非严格，this 都是 obj

function User(name) {
  this.name = name;
}
new User("Alice"); // this 始终指向新实例
```
**结论**：只要调用时左侧有明确的对象（`obj.fn()`）或使用 `new`，两种模式行为完全一致。

#### 4. 箭头函数（词法继承）
```js
const obj = {
  id: 1,
  log: function() {
    setTimeout(() => {
      console.log(this.id); // this 继承自 log 方法调用时的 obj
    }, 0);
  }
};
obj.log(); // 输出 1（严格/非严格结果相同）
```
箭头函数**没有自己的 `this`**，它捕获的是定义时外层作用域的 `this`。外层是严格还是非严格，决定了它最终拿到的是 `undefined` 还是具体对象。

#### 5. DOM 事件回调
```js
btn.addEventListener("click", function() {
  console.log(this); // 永远指向 btn 元素
});
```
浏览器底层在派发事件时，会强制将回调函数的 `this` 绑定到事件目标元素。**严格模式无法覆盖此行为**。

---

### 💡 三大核心差异总结
1. **默认回退策略不同**  
   非严格：`this` 找不到宿主 → 回退到全局对象（`window`）  
   严格：`this` 找不到宿主 → 直接为 `undefined`

2. **显式绑定 `null/undefined` 时不转换**  
   非严格会“自作主张”转成全局对象；严格模式保持原值，更符合函数式编程直觉。

3. **错误暴露时机提前**  
   严格模式下，`this` 为 `undefined` 时若尝试访问属性（如 `this.xxx`），会立即抛 `TypeError`，而非非严格模式下的 `undefined.xxx` 静默失败或后续难以追踪的 Bug。

---

### 🛠 现代开发建议
- **新项目无需手动处理**：ES6 模块、Vue/React 组件、Class、TS 编译输出均默认运行在严格模式，`this` 行为已统一为严格规则。
- **避免依赖 `this` 隐式绑定**：在回调、定时器、解构赋值中，优先使用**箭头函数**或显式 `.bind(this)`。
- **老代码兼容**：若维护非严格老代码，可通过 ESLint 规则 `@typescript-eslint/no-this-alias` 或 `consistent-this` 提前拦截 `this` 滥用。

> ✅ 一句话记忆：**严格模式下，没有明确调用对象的 `this` 一律是 `undefined`；有对象的（方法/构造/事件/箭头继承）规则不变。**