# 14 "switch" 语句

> 原文：`zh.javascript.info/1-js/02-first-steps/14-switch/article.md`

## 知识点

### 1. 语法
- `switch` 可替代多个 `if` 判断，为多分支选择提供更具描述性的写法。
- 至少一个 `case` 块，可选一个 `default` 块：

```js
switch(x) {
  case 'value1':  // if (x === 'value1')
    ...
    [break]

  case 'value2':  // if (x === 'value2')
    ...
    [break]

  default:
    ...
    [break]
}
```

- 执行规则：
  1. 用**严格相等 `===`** 依次比较 `x` 与各 `case` 的值。
  2. 相等则执行该 `case` 下的代码，**直到遇到最近的 `break`**（或 `switch` 末尾）。
  3. 没有匹配则执行 `default`（如果存在）。

### 2. 举个例子

```js
let a = 2 + 2;

switch (a) {
  case 3:
    alert( 'Too small' );
    break;
  case 4:
    alert( 'Exactly!' );   // 匹配，执行到 break
    break;
  case 5:
    alert( 'Too big' );
    break;
  default:
    alert( "I don't know such values" );
}
```

**没有 `break` 会"穿透"（fall-through）**：程序不做任何检查，继续执行下一个 `case`。

```js
let a = 2 + 2;

switch (a) {
  case 3:
    alert( 'Too small' );
  case 4:
    alert( 'Exactly!' );
  case 5:
    alert( 'Too big' );
  default:
    alert( "I don't know such values" );
}
// 从 case 4 开始连续执行三个 alert：'Exactly!'、'Too big'、"I don't know such values"
```

**任何表达式都可以作为 `switch/case` 的参数**：

```js
let a = "1";
let b = 0;

switch (+a) {
  case b + 1:
    alert("this runs, because +a is 1, exactly equals b+1");
    break;
  default:
    alert("this doesn't run");
}
```

### 3. `case` 分组
- 共享同一段代码的多个 `case` 可以写在一起：

```js
let a = 3;

switch (a) {
  case 4:
    alert('Right!');
    break;

  case 3: // 3 和 5 分为一组
  case 5:
    alert('Wrong!');
    alert("Why don't you take a math class?");
    break;

  default:
    alert('The result is strange. Really.');
}
```

- 分组能力其实是"没有 `break` 就穿透"的副作用：`case 3` 匹配后没有 `break`，直接执行到 `case 5` 的代码。

### 4. 类型很关键
- `switch` 用的是**严格相等**，类型不同就不匹配：

```js
let arg = prompt("Enter a value?");
switch (arg) {
  case '0':
  case '1':
    alert( 'One or zero' );
    break;

  case '2':
    alert( 'Two' );
    break;

  case 3:
    alert( 'Never executes!' ); // prompt 返回字符串 "3"，不 === 数字 3
    break;
  default:
    alert( 'An unknown value' );
}
```

- 输入 `3` 时走 `default`，`case 3` 是一段永远不会执行的死代码。要匹配数字得先 `Number(arg)` 或 `+arg`。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `switch` 只接受 `int`/`char`/`String`/枚举等 | 接受**任意类型任意表达式** | `case` 后面也可以是表达式（`case b + 1:`），Java 要求编译期常量 |
| `String` 用 `equals` 比较 | 用 `===` 比较 | 类型不同直接不匹配，`"3"` 不等于 `3` |
| 传统 `switch` 同样有 fall-through，需要 `break` | 相同 | 语义一致，`case` 分组写法也一样 |
| Java 14+ 有 `switch` 表达式和箭头 `case X ->`，无穿透且可返回值 | 没有 `switch` 表达式 | JS 的 `switch` 只是语句，要取值只能在 `case` 里赋值或 `return` |
| Java 21 有模式匹配 `case Integer i ->` | 无 | 需要按类型/范围分支时用 `if..else if` |
| 对 `null` 做 `switch` 抛 NPE（Java 21 前） | `switch (null)` 不报错，走 `default` | `case null:` 在 JS 里是合法的 |

**实践建议**：
- 从 `prompt`、表单、URL 拿到的值都是字符串，`switch` 前先明确转换类型，或者 `case` 也写字符串。
- 每个 `case` 结尾写 `break`（有意分组时除外），末尾加 `default` 处理意外值。
- 简单的"值 → 结果"映射用对象字面量查表（`const map = { a: 1, b: 2 }; map[key]`）往往比 `switch` 更简洁，后续学对象时可以体会。

## 一句话总结
`switch` 语法和穿透规则与 Java 传统 `switch` 一致，但比较用严格相等 `===`（类型必须相同），且 `switch` 和 `case` 后可以是任意类型的任意表达式；没有 Java 14+ 的 `switch` 表达式形式。
