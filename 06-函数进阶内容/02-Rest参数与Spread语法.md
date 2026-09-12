# 02 Rest 参数与 Spread 语法

> 原文：`zh.javascript.info/1-js/06-advanced-functions/02-rest-parameters-spread/article.md`
> **ES6 核心语法**，`...` 两个方向的用法，日常代码中无处不在。

## 知识点

### 1. Rest 参数 `...`（收集 → 数组）
- JS 调用函数时传多少参数都**不报错**（多余的被忽略），不像 Java 严格匹配签名。
- 用 `...变量名` 把剩余参数收集成**真数组**：

```js
function sumAll(...args) {
  let sum = 0;
  for (let arg of args) sum += arg;
  return sum;
}
sumAll(1, 2, 3); // 6

// 前几个参数单独命名，剩余的收集：
function showName(firstName, lastName, ...titles) { ... }
```

- ⚠️ **`...rest` 必须在参数列表末尾**，`function f(a, ...rest, b)` 是语法错误。

### 2. `arguments` 变量（老式写法，了解即可）
- 普通函数内可用的特殊**类数组对象**，按索引存所有参数、可迭代。
- 缺点：不是真数组，**没有数组方法**（`arguments.map(...)` 报错）；只能获取全部参数，不能只取一部分。
- ⚠️ **箭头函数没有自己的 `arguments`**，在箭头函数里访问 `arguments` 拿到的是外层普通函数的。
- 结论：**需要参数数组就用 rest 参数**，`arguments` 只在老代码里见。

### 3. Spread 语法 `...`（展开 → 参数列表/字面量）
与 rest 参数长得像，用途**正好相反**——把可迭代对象展开：

```js
let arr = [3, 5, 1];
Math.max(...arr);              // 5，等价于 Math.max(3, 5, 1)
Math.max(...arr1, ...arr2);    // 可以展开多个
Math.max(1, ...arr1, 25);      // 可以和普通值混用

let merged = [0, ...arr, 2, ...arr2]; // 合并数组
[..."Hello"];                  // ['H','e','l','l','o']，对任何可迭代对象有效
```

- `Math.max(arr)` 直接传数组得到 `NaN` —— 这是 spread 要解决的典型问题。
- 内部走迭代器协议（和 `for..of` 一样），所以**只能用于可迭代对象**。

### 4. `[...obj]` vs `Array.from(obj)`
- `Array.from`：可迭代对象**和**类数组对象都行。
- spread：只认**可迭代对象**。
- 结论：把"东西"转数组时 `Array.from` 更通用。

### 5. 浅拷贝数组/对象（高频）
```js
let arrCopy = [...arr];     // 比 Object.assign([], arr) 简洁
let objCopy = { ...obj };   // 比 Object.assign({}, obj) 简洁
arr === arrCopy;            // false（不同引用），内容是浅拷贝
```

### 6. 如何区分 rest 和 spread
- `...` 出现在**函数参数列表末尾** → rest 参数（收集）。
- `...` 出现在**函数调用或数组/对象字面量中** → spread（展开）。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 可变参数 `void f(String... args)` | rest 参数 `function f(...args)` | 几乎一样，JS 的 rest 拿到的是真数组，比 Java 隐式数组更直白 |
| 调用时参数个数必须匹配签名 | 传多传少都不报错 | JS 函数调用极其宽松，这是 rest 存在的背景 |
| 无对应语法 | spread：`Math.max(...arr)`、`[...arr1, ...arr2]` | Java 要把数组展开传给可变参数**直接传数组即可**（自动匹配），JS 必须显式 `...` |
| `Arrays.copyOf` / `new ArrayList<>(list)` | `[...arr]` / `{...obj}` | JS 浅拷贝一行搞定（注意是浅拷贝） |
| 无 | 字符串展开 `[...str]` | 依赖迭代器协议，Java 无对应 |

## 一句话总结
`...` 出现在参数列表末尾是 **rest**（把剩余参数收集成真数组，必须放最后），出现在调用处或字面量里是 **spread**（把任何可迭代对象展开，可合并数组 `[...a, ...b]`、浅拷贝 `[...arr]`/`{...obj}`、把数组传给 `Math.max(...arr)`）；老式 `arguments` 是类数组且箭头函数没有它，新代码统一用 rest 参数。
