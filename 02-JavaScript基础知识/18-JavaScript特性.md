# 18 JavaScript 特性

> 原文：`zh.javascript.info/1-js/02-first-steps/18-javascript-specials/article.md`
> 本篇是对 01–17 的复习总结，特别关注细节和陷阱。每节末尾标注对应的笔记文件。

## 知识点

### 1. 代码结构
- 语句用分号分隔：`alert('Hello'); alert('World');`
- 换行通常也被视为分隔符（**自动分号插入**），但有时不起作用：

```js
alert("There will be an error after this message")

[1, 2].forEach(alert) // 会被解析为 alert(...)[1, 2].forEach(alert)，报错
```

- 大多数代码风格指南要求**每个语句后都加分号**。
- 代码块 `{...}` 及带代码块的语法结构（函数声明、循环）后不需要分号；多加了分号也不是错误，会被忽略。
- → `02-代码结构.md`

### 2. 严格模式
- 脚本顶部（或函数体开头）写 `'use strict';` 完全启用现代 JS 特性。
- 没有它一切仍能工作，但某些功能以老式"兼容"方式运行。
- 一些现代特性（如 class、模块）会隐式启用严格模式。
- → `03-现代模式-use-strict.md`

### 3. 变量
- 声明方式：`let`、`const`（不可重新赋值）、`var`（旧式）。
- 变量名：字母、数字（首字符不能是数字）、`$`、`_`；非拉丁字母允许但不常用。
- **动态类型**：`let x = 5; x = "John";`
- 8 种数据类型：

| 类型 | 说明 |
|---|---|
| `number` | 浮点数和整数 |
| `bigint` | 任意长度整数 |
| `string` | 字符串 |
| `boolean` | `true` / `false` |
| `null` | 单个值 `null`，表示"空"或"不存在" |
| `undefined` | 单个值 `undefined`，表示"未分配" |
| `object`、`symbol` | 复杂数据结构和唯一标识符，后续学习 |

- `typeof` 返回类型，两个例外：

```js
typeof null == "object"         // 语言设计错误
typeof function(){} == "function" // 函数被特殊对待
```

- → `04-变量.md`、`05-数据类型.md`

### 4. 交互
| 函数 | 作用 | 返回值 |
|---|---|---|
| `prompt(question[, default])` | 提问并让用户输入 | 输入的字符串，取消则 `null` |
| `confirm(question)` | 让用户选择确定/取消 | `true` / `false` |
| `alert(message)` | 输出消息 | 无 |

- 都是**模态框**：暂停代码执行，阻止用户与页面其他部分交互。

```js
let userName = prompt("Your name?", "Alice");
let isTeaWanted = confirm("Do you want some tea?");

alert( "Visitor: " + userName );      // Alice
alert( "Tea wanted: " + isTeaWanted ); // true
```

- → `06-交互-alert-prompt-confirm.md`

### 5. 运算符
- **算术**：`+ - * /`、取余 `%`、幂 `**`。二元 `+` 遇字符串做拼接，任一操作数是字符串则另一个也转成字符串：

```js
alert( '1' + 2 ); // '12'
alert( 1 + '2' ); // '12'
```

- **赋值**：`a = b`，复合赋值 `a *= 2`。
- **按位运算**：把操作数当 32 位整数在位级上操作。
- **三元**：`cond ? resultA : resultB`，唯一有三个操作数的运算符。
- **逻辑**：`&&`、`||` 短路求值，**返回运算停止处的值**（不一定是布尔）；`!` 转布尔后取反。
- **空值合并**：`a ?? b`，`a` 为 `null/undefined` 时取 `b`，否则取 `a`。
- **比较**：
  - `==` 对不同类型转成数字比较（`null` 和 `undefined` 例外：它们只彼此相等）：

```js
alert( 0 == false ); // true
alert( 0 == '' );    // true
```

  - `===` 不做转换，类型不同即不相等。
  - `null` 和 `undefined` 只在 `==` 下相等，不等于任何其他值。
  - 大于/小于比较字符串时按字符逐个比较，其他类型转数字。
- **其他**：逗号运算符等。
- → `08-基础运算符-数学运算.md`、`09-值的比较.md`、`11-逻辑运算符.md`、`12-空值合并运算符.md`

### 6. 循环
- 三种循环：

```js
while (condition) { ... }

do { ... } while (condition);

for (let i = 0; i < 10; i++) { ... }
```

- `for (let ...)` 内声明的变量只在循环内可见；也可省略 `let` 重用已有变量。
- `break` 退出整个循环，`continue` 跳过当前迭代；用**标签**跳出嵌套循环。
- 处理对象的其他循环（`for..in`、`for..of`）后续学习。
- → `13-循环-while和for.md`

### 7. `switch` 结构
- 替代多个 `if` 检查，内部用 **`===` 严格相等**比较：

```js
let age = prompt('Your age?', 18);

switch (age) {
  case 18:
    alert("Won't work"); // prompt 返回的是字符串，不是数字
    break;

  case "18":
    alert("This works!");
    break;

  default:
    alert("Any value not equal to one above");
}
```

- → `14-switch语句.md`

### 8. 函数
三种创建方式：

```js
// 1. 函数声明：主代码流中的函数
function sum(a, b) {
  let result = a + b;
  return result;
}

// 2. 函数表达式：表达式上下文中的函数
let sum = function(a, b) {
  let result = a + b;
  return result;
};

// 3. 箭头函数
let sum = (a, b) => a + b;          // 表达式在右侧，自动返回

let sum = (a, b) => {               // 多行语法，需要 return
  // ...
  return a + b;
};

let sayHi = () => alert("Hello");   // 没有参数
let double = n => n * 2;            // 一个参数
```

- 函数可以有局部变量（函数内声明的或参数列表中的），只在函数内可见。
- 参数可以有默认值：`function sum(a = 1, b = 2) {...}`
- 函数**总是**返回一些东西，没有 `return` 就返回 `undefined`。
- → `15-函数.md`、`16-函数表达式.md`、`17-箭头函数基础.md`

## 与 Java 对比（本部分 18 篇的差异总表）
| 主题 | Java | JS | 详见 |
|---|---|---|---|
| 类型系统 | 静态类型，编译期检查 | 动态类型，`typeof` 运行时检查 | 04、05 |
| 数值 | `int`/`long`/`double` 等多种 | 只有 `number`（双精度）+ `bigint` | 05 |
| 空值 | 只有 `null` | `null` 和 `undefined` 两种 | 05、09、12 |
| 字符串 | `char` + `String`，`equals` 比较 | 只有 `string`，`===` 比内容，模板字符串 `` `${x}` `` | 05、09 |
| 类型转换 | 显式，失败抛异常 | 大量隐式转换，`Number()` 失败返回 `NaN` 不抛异常 | 07 |
| 除法 | 整数除法截断 | 永远是浮点除法 | 08 |
| 字符串拼接 | `+` 拼接，其他运算符对字符串报错 | `+` 拼接，`- * /` 把字符串转数字 | 08 |
| 相等 | `==` 比引用/值，`equals` 比内容 | `==` 宽松相等（转换），`===` 严格相等；**默认用 `===`** | 09 |
| 条件 | 必须是 `boolean` | 任意值转布尔，5 个假值：`0`、`""`、`null`、`undefined`、`NaN` | 07、10 |
| 逻辑运算符 | 返回 `boolean` | 返回操作数原始值，`\|\|` 取第一个真值，`&&` 取第一个假值 | 11 |
| 默认值 | `x != null ? x : def` / `Optional` | `x ?? def`（推荐）或 `x \|\| def`（会替换 `0`/`""`） | 12 |
| `switch` | 支持 `String`/枚举，Java 14+ 有 switch 表达式 | 任意类型任意表达式，`===` 比较，无 switch 表达式 | 14 |
| 函数 | 方法必须属于类，参数个数严格 | 独立函数，是一等值；参数缺省为 `undefined` 不报错；有默认参数 | 15、16 |
| Lambda | 需函数式接口 | 箭头函数，无需接口，对象字面量返回要加括号 | 17 |
| 分号 | 必须 | 可自动插入，但建议始终加 | 02、18 |
| 严格模式 | 无此概念 | `'use strict'`，现代代码必开 | 03 |

**实践建议（全篇汇总）**：
- 默认 `const`，需要重新赋值用 `let`，不用 `var`；文件顶部 `'use strict'`（或用模块/class 隐式启用）。
- 一律 `===`/`!==`；判断 `null` 和 `undefined` 可用 `x == null` 或 `x ?? def`。
- 从 `prompt`/表单/URL 拿到的都是字符串，用之前 `Number()` 或一元 `+` 转换并检查 `NaN`。
- 默认值优先 `??` 而不是 `||`。
- 回调和单行函数用箭头函数，主流程函数用函数声明。
- 每条语句加分号，`return` 后不换行。

## 一句话总结
本部分学完了 JS 的代码结构、严格模式、变量与 8 种类型、交互函数、运算符（含 `??`）、比较（`===`）、条件与循环、`switch`、三种函数写法；对 Java 开发者来说最需要转变的认知是：动态类型 + 大量隐式转换、两种空值、`==` 与 `===` 的区别、truthy/falsy、逻辑运算符返回原始值、函数是一等值。
