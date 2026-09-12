# 06 交互：alert、prompt 和 confirm

> 原文：`zh.javascript.info/1-js/02-first-steps/06-alert-prompt-confirm/article.md`

## 知识点

### 1. `alert`
- 显示一条信息，等待用户点"确定"。

```js
alert("Hello");
```

- 弹出的小窗口叫**模态窗（modal）**：在用户处理完窗口之前，不能与页面其他部分交互，脚本执行也被暂停。

### 2. `prompt`
- 语法：`result = prompt(title, [default]);`
  - `title`：显示给用户的文本。
  - `default`：可选，input 框的初始值。语法中的方括号 `[...]` 表示参数可选。
- 显示带文本消息、输入框、确定/取消按钮的模态窗。
- **返回值**：点确定返回用户输入的文本（**字符串**）；点取消或按 `Esc` 返回 `null`。

```js
let age = prompt('How old are you?', 100);

alert(`You are ${age} years old!`); // You are 100 years old!
```

- IE 的坑：不传第二个参数时 IE 会在输入框里插入 `"undefined"`，所以建议始终提供第二个参数：`prompt("Test", '')`。

### 3. `confirm`
- 语法：`result = confirm(question);`
- 显示带 `question` 和确定/取消两个按钮的模态窗。
- **返回值**：确定 → `true`，取消或 `Esc` → `false`。

```js
let isBoss = confirm("Are you the boss?");

alert( isBoss ); // 点"确定"显示 true
```

### 4. 三个函数的共同特点与限制
| 函数 | 作用 | 返回值 |
|---|---|---|
| `alert(msg)` | 显示信息 | 无（`undefined`） |
| `prompt(title, default)` | 要求输入文本 | 字符串 / 取消时 `null` |
| `confirm(question)` | 让用户确认或取消 | `true` / `false` |

- 都是**模态**的：暂停脚本执行，阻止用户与页面其余部分交互。
- 两个限制：
  1. 窗口位置由浏览器决定（通常页面中央），不可控。
  2. 窗口外观由浏览器决定，无法修改样式。
- 这是"简单"的代价；需要更漂亮的交互时用其他方式（自定义 DOM 弹窗），但不追求花哨时用这三个就够了。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `System.out.println` / `JOptionPane.showMessageDialog` | `alert` | `alert` 是**浏览器**提供的函数，不是 JS 语言本身的一部分；Node.js 里没有 |
| `Scanner.nextLine()` / `JOptionPane.showInputDialog` | `prompt` | 返回值永远是**字符串**或 `null`，要当数字用需要显式转换（下一篇讲） |
| `JOptionPane.showConfirmDialog` 返回 `int` 常量 | `confirm` 返回 `boolean` | JS 直接返回 `true/false`，更直接 |
| 控制台程序输入是阻塞式 | 三者都是模态阻塞式 | 脚本会停在那一行等用户操作，与页面 JS 的异步风格不同，实际项目中几乎不用 |

**实践建议**：
- 这三个函数只适合学习和快速调试，正式项目中用 `console.log` 调试、用 DOM/框架组件做交互。
- `prompt` 返回值记得处理 `null`（用户取消）和字符串类型两种情况。

## 一句话总结
`alert` 显示信息、`prompt` 获取输入（返回字符串或 `null`）、`confirm` 获取确认（返回布尔值）；三者都是浏览器提供的模态窗，简单但样式和位置不可控。
