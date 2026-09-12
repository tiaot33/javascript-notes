# 13 循环：while 和 for

> 原文：`zh.javascript.info/1-js/02-first-steps/13-while-for/article.md`

## 知识点

### 0. 概述
- 本篇只讲基础循环：`while`、`do..while`、`for(..; ..; ..)`。
- 遍历对象属性的 `for..in`、遍历数组和可迭代对象的 `for..of` 在后续章节（对象、数组、iterables）讲。

### 1. `while` 循环
- 语法：`while (condition) { 循环体 }`，条件为真时重复执行循环体。
- 循环体单次执行叫**一次迭代**。

```js
let i = 0;
while (i < 3) { // 依次显示 0、1、2
  alert( i );
  i++;
}
```

- 没有 `i++` 就是死循环；浏览器有机制阻止，服务端可终止进程。
- 条件可以是任何表达式或变量，会被转成布尔值。`while (i != 0)` 可简写为 `while (i)`：

```js
let i = 3;
while (i) { // i 变成 0 时条件为假，终止
  alert( i );
  i--;
}
```

- 循环体只有一条语句时可省略大括号：`while (i) alert(i--);`

### 2. `do..while` 循环
- 条件检查移到循环体**下面**，先执行一次再检查：

```js
let i = 0;
do {
  alert( i );
  i++;
} while (i < 3);
```

- 很少使用，只在希望**不管条件如何循环体至少执行一次**时用；通常更倾向于 `while(…) {…}`。

### 3. `for` 循环
- 最常用的循环形式：`for (begin; condition; step) { 循环体 }`

```js
for (let i = 0; i < 3; i++) { // 0、1、2
  alert(i);
}
```

| 语句段 | 示例 | 何时执行 |
|---|---|---|
| begin | `let i = 0` | 进入循环时执行一次 |
| condition | `i < 3` | 每次迭代**之前**检查，为 false 则停止 |
| body | `alert(i)` | 条件为真时重复运行 |
| step | `i++` | 每次循环体迭代**之后**执行 |

- 执行流程：`begin` 一次 → (检查 condition → body → step) → (检查 condition → body → step) → …

**内联变量声明**：在 `for` 里用 `let` 声明的计数变量只在循环内可见：

```js
for (let i = 0; i < 3; i++) {
  alert(i); // 0, 1, 2
}
alert(i); // 错误，没有这个变量
```

- 也可以用循环外已有的变量，此时循环结束后变量仍可见：

```js
let i = 0;
for (i = 0; i < 3; i++) { ... }
alert(i); // 3
```

**省略语句段**：`for` 的任何语句段都可以省略，但**两个 `;` 必须存在**。

```js
let i = 0;
for (; i < 3; i++) { alert( i ); } // 省略 begin

for (; i < 3;) { alert( i++ ); }   // 省略 step，等价于 while (i < 3)

for (;;) { /* 无限循环 */ }
```

### 4. 跳出循环：`break`
- 随时强制退出循环，控制权交给循环后的第一行。
- "无限循环 + `break`"适合条件检查不在循环开头/结尾、而在中间甚至多个位置的场景：

```js
let sum = 0;

while (true) {
  let value = +prompt("Enter a number", '');
  if (!value) break; // 输入空行或取消时退出
  sum += value;
}
alert( 'Sum: ' + sum );
```

### 5. 继续下一次迭代：`continue`
- `break` 的"轻量版"：不停止整个循环，只结束**当前这次**迭代，进入下一次（条件允许的话）。

```js
for (let i = 0; i < 10; i++) {
  if (i % 2 == 0) continue; // 偶数跳过剩余部分
  alert(i); // 1, 3, 5, 7, 9
}
```

- **`continue` 利于减少嵌套**：不用 `continue` 就得把逻辑包进 `if` 块，多一层嵌套；`if` 内代码多行时可读性下降。

**禁止 `break/continue` 出现在 `?` 的右边**：`break`/`continue` 是语句不是表达式，不能用在三元运算符里，会报语法错误。

```js
(i > 5) ? alert(i) : continue; // SyntaxError
```

这也是不建议用 `?` 替代 `if` 的又一个理由。

### 6. `break/continue` 标签
- 需要一次跳出**多层嵌套**循环时，普通 `break` 只能跳出内层，用**标签**解决。
- 标签是循环前带冒号的标识符：`labelName: for (...) { ... }`
- `break <labelName>` 跳出到标签所在的循环之外：

```js
outer: for (let i = 0; i < 3; i++) {

  for (let j = 0; j < 3; j++) {

    let input = prompt(`Value at coords (${i},${j})`, '');

    if (!input) break outer; // 空字符串或取消 → 跳出两层循环

    // 用得到的值做些事……
  }
}

alert('Done!'); // break outer 后直接到这里
```

- 标签可以单独一行：`outer:\nfor (...) { ... }`
- `continue <labelName>` 跳到标签循环的下一次迭代。
- **标签不是 goto**：不能跳到代码任意位置，`break label` 必须在被标记的代码块内部；技术上任何被标记的代码块都可以 `break`（`label: { ... break label; ... }`），但 99.9% 用在循环里；`continue` 只能在循环内。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `while`、`do..while`、`for(;;)` 语法相同 | 相同 | 完全一致 |
| `for (int i = 0; ...)` | `for (let i = 0; ...)` | 用 `let`，作用域同样限于循环内；**不要用 `var`**（`var` 会泄漏到函数作用域） |
| `while (cond)` 中 `cond` 必须是 `boolean` | 任意值，做布尔转换 | `while (i)`、`while (str)` 合法 |
| `break`、`continue` 相同 | 相同 | 完全一致 |
| 标签 `outer: for (...) { break outer; }` | 相同 | 语法一模一样，Java 的标签也只能用于 `break`/`continue` |
| 增强 for：`for (T x : collection)` | `for..of`（数组/可迭代）、`for..in`（对象属性） | 后续章节讲；**注意 `for..in` 遍历的是 key，不要用它遍历数组** |
| `for (;;)` 无限循环合法 | 相同 | 同样惯用 `while (true)` |

**实践建议**：
- 计数循环用 `for (let i = ...)`，永远不用 `var`。
- 后面学了数组后，遍历数组优先用 `for..of` 或数组方法（`forEach`/`map`），而不是索引 `for`。
- 标签 `break` 与 Java 一样合法但少见，能拆成函数 `return` 的话更清晰。

## 一句话总结
`while`、`do..while`、`for`、`break`、`continue`、标签跳出多层循环——语法和语义与 Java 完全一致；唯二差别是循环条件会做布尔转换，以及计数变量要用 `let` 声明。
