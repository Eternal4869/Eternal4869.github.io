手写 `apply` 和 `call` 的底层逻辑 **90% 是完全相同的**，唯一区别在于 **参数传递形式**。

---
### 📦 完整实现（面试标准版）
```javascript
Function.prototype.myApply = function(context, argsArray) {
  // 1. 类型守卫
  if (typeof this !== 'function') {
    throw new TypeError('myApply 只能被函数调用');
  }

  // 2. 规范化上下文（与 call 完全一致）
  const ctx = context == null ? globalThis : Object(context);

  // 3. 生成唯一键，防止覆盖原对象属性
  const fnKey = Symbol('myApplyFn');

  // 4. 临时挂载
  ctx[fnKey] = this;

  // 5. 【核心差异】参数是数组/类数组，展开后执行
  const args = argsArray == null ? [] : Array.from(argsArray);
  const result = ctx[fnKey](...args);

  // 6. 清理现场
  delete ctx[fnKey];

  // 7. 返回结果
  return result;
};
```

---
### 🔍 逐行拆解（重点看第 5 步）

| 代码片段 | 通俗解释 | 与 `call` 的差异 |
|:---|:---|:---|
| `function(context, argsArray)` | `apply` 只接收两个参数：上下文 + **参数数组**。 | `call` 是 `function(context, ...args)`，接收无限个独立参数。 |
| `const args = argsArray == null ? [] : Array.from(argsArray);` | 把传入的数组、类数组（如 `arguments`、`NodeList`）或可迭代对象安全转为真数组。如果没传或传 `null`，默认为空数组 `[]`。 | `call` 不需要这步，直接用 `...args` 收集即可。 |
| `ctx[fnKey](...args);` | 将数组展开成独立参数传入函数。因为函数现在是 `ctx` 的属性，执行时 `this` 自然指向 `ctx`。 | `call` 是 `ctx[fnKey](...args)`，但 `args` 是 rest 参数收集来的独立值，不是数组。 |
| 其余部分（守卫、上下文处理、Symbol、清理、返回） | **与 `call` 完全一致**，不再赘述。 | 无差异。 |

---
### 💡 为什么这么设计？关键细节

1. **为什么用 `Array.from(argsArray)` 而不是直接 `...argsArray`？**  
   原生 `apply` 支持传入 **类数组对象**（如 `function` 的 `arguments`、DOM 的 `NodeList`）。直接展开 `...argsArray` 在严格模式下对非数组对象可能报错。`Array.from()` 能安全兼容数组、类数组、Set/Map 等可迭代对象，更贴近规范。

2. **`argsArray` 传了非数组会怎样？**  
   原生 `apply` 会直接抛 `TypeError`。面试中写出 `Array.from` 已足够体现严谨性。若追求 100% 还原原生行为，可加一层类型判断：
   ```javascript
   if (argsArray != null && typeof argsArray.length === 'undefined') {
     throw new TypeError('apply 第二个参数必须是数组或类数组');
   }
   ```

3. **性能考量**  
   `Symbol` 创建和 `delete` 属性在现代 V8 引擎中开销极小。若追求极致性能，可用 `Math.random().toString(36).slice(2)` 替代 `Symbol`（兼容性更好但极低概率冲突），面试中 `Symbol` 是首选。

---
### 🌍 Java 视角对照：`call` vs `apply` vs 反射

| 特性 | JavaScript `call` | JavaScript `apply` | Java 反射 `Method.invoke` |
|:---|:---|:---|:---|
| 调用形式 | `func.call(obj, a, b)` | `func.apply(obj, [a, b])` | `method.invoke(obj, a, b)` 或 `method.invoke(obj, new Object[]{a, b})` |
| 参数形态 | 独立参数列表 | 打包成数组再解包 | 底层统一按 `Object[]` 处理，上层重载支持变长参数 |
| 本质 | 给函数临时绑定 `this` | 同上 | Java 的 `this` 是实例引用，无需动态绑定；反射只是“绕过编译期检查调用方法” |

**一句话类比：**  
> Java 的方法调用像“固定插座”（插头和插座出厂就配对好）。  
> JS 的函数像“万能插头”，`call` 是 `func.call(插座, 线1, 线2)`，`apply` 是 `func.apply(插座, [线1, 线2])`。插法不同，通电效果完全一样。

---
### ✅ 测试用例（验证边界）
```javascript
function add(x, y) {
  return this.base + x + y;
}

const obj = { base: 10 };

console.log(add.myApply(obj, [5, 8]));          // 23
console.log(add.myApply(null, [1, 2]));         // 非严格模式指向全局
console.log(add.myApply(obj, new Set([3, 4]))); // 7 (Array.from 兼容可迭代对象)

// 模拟 arguments 传参
function wrapper() {
  return add.myApply(obj, arguments); // arguments 是类数组
}
console.log(wrapper(6, 7)); // 23
```

---
### 📌 总结
- `apply` 和 `call` 是 **双胞胎**，底层都是“借属性挂载 → 改变 this 指向 → 执行 → 清理”。
- 唯一区别：`call` 适合 **参数个数固定** 的场景；`apply` 适合 **参数来自数组/类数组** 的场景。
- 现代 JS 中，两者几乎可被 `func.bind(context)(...args)` 或 `Reflect.apply(func, context, argsArray)` 替代，但手写实现是理解 JS 运行时 `this` 机制的最佳切入点。