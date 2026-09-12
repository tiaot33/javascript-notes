# 04 老旧的 "var"

> 原文：`zh.javascript.info/1-js/06-advanced-functions/04-var/article.md`
> **新代码一律用 `let/const`**；学 `var` 只为看懂旧脚本和面试题。
> ⚠️ 注意：JS 的 `var` 与 Java 10 的 `var`（局部变量类型推断）**毫无关系**，别混淆。

## 知识点

### 1. `var` 没有块级作用域
- `var` 声明的变量只有**函数作用域**和**全局作用域**，直接"穿透" `if`、`for` 等代码块：

```js
if (true) {
  var test = true;
}
alert(test); // true（let 的话这里 ReferenceError）

for (var i = 0; i < 10; i++) { ... }
alert(i); // 10，循环结束后仍可见
```

- 原因：早期 JS 中代码块没有词法环境，`var` 是那个时代的遗留。

### 2. `var` 允许重复声明
```js
var user = "Pete";
var user = "John"; // 不报错，第二次声明被忽略（赋值生效）
alert(user); // John
```
同样的代码用 `let` → `SyntaxError: 'user' has already been declared`。

### 3. 提升（hoisting）：声明提前，赋值不提前
- 函数开始时就会处理所有 `var` 声明（全局脚本启动时对应全局变量），与书写位置无关：

```js
function sayHi() {
  phrase = "Hello";
  alert(phrase);
  var phrase;      // 声明被提升到函数开头
}
```

- 即使在永远不执行的分支里，声明也会被处理：

```js
function sayHi() {
  phrase = "Hello";
  if (false) {
    var phrase;    // 永远不执行，但声明照样提升
  }
  alert(phrase);   // "Hello"
}
```

- **关键规则：只有声明被提升，赋值留在原地**：

```js
function sayHi() {
  alert(phrase);        // undefined（存在但未赋值，不报错！）
  var phrase = "Hello"; // 等价于：var phrase;（开头）+ phrase = "Hello"（此处）
}
```

- 对比 `let`：声明前访问直接 `ReferenceError`（暂时性死区），`var` 声明前访问得到 `undefined`。

### 4. IIFE（立即调用函数表达式）
旧时代没有块级作用域，程序员用 IIFE 模拟私有作用域：

```js
(function() {
  var message = "Hello"; // 私有，外部不可见
  alert(message);
})();
```

- 必须用括号包住 `function`：否则 JS 把它当**函数声明**解析——声明必须有名字，且不能立即调用。
- 其他让它成为"表达式"的写法：`!function(){}()`、`+function(){}()`。
- **现在有了 `let` 的块级作用域和模块，IIFE 已被淘汰**，只需能读懂。

## 与 Java 对比
| Java | JS (`var`) | 说明 |
|---|---|---|
| 变量先声明后使用，否则编译错误 | 声明提升，使用前值为 `undefined` | JS `var` 不报错的"宽容"是 bug 温床 |
| 块级作用域（`if`/`for` 内变量外不可见） | 无块级作用域，只有函数/全局作用域 | `for (var i=0;...)` 的 `i` 会泄漏到循环外 |
| 重复声明编译错误 | `var` 重复声明静默忽略 | 隐蔽错误的来源 |
| 无 IIFE 概念 | `(function(){...})()` | 相当于手写一个立即执行的匿名方法来造私有作用域，Java 直接 `{}` 块就有作用域 |
| Java 10 `var` = 类型推断 | JS `var` = 老式声明关键字 | **同名完全不同**，Java 的 `var` 编译期强类型，JS 的 `var` 是历史遗留 |

## 一句话总结
`var` 与 `let/const` 两大区别：**①没有块级作用域**（穿透 `if`/`for`，只有函数/全局作用域）**②声明被提升**到函数开头处理（赋值不提升，声明前访问得 `undefined` 而非报错），且允许重复声明；旧代码用 IIFE `(function(){...})()` 模拟块级作用域——新代码全部用 `let/const`，学 `var` 只为读旧脚本。
