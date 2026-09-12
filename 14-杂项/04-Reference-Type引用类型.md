# 04 Reference Type（引用类型）

> 原文：`zh.javascript.info/1-js/99-js-misc/04-reference-type/article.md`
> 深入语言内部的"锦上添花"知识：解释 `obj.method()` 的 `this` 到底是怎么传过去的——以及为什么动态取方法会丢 `this`。

## 知识点

### 1. 引子：动态方法调用丢 `this`
```js
let user = {
  name: "John",
  hi() { alert(this.name); },
  bye() { alert("Bye"); }
};

user.hi();                                    // 正常
(user.name == "John" ? user.hi : user.bye)(); // ❌ Error！this 变成 undefined
```
三元表达式选出了 `user.hi`，紧接着 `()` 调用——却失败了。要理解原因，得拆开看 `obj.method()` 的本质。

### 2. `obj.method()` 其实是两个操作
1. 点 `.` 取出属性 `obj.method` 的值；
2. `()` 执行它。

`this` 的信息怎么从第 1 步传到第 2 步？如果拆开两行写，`this` 必丢：
```js
let hi = user.hi; // 只拿走了函数本身
hi();             // ❌ this = undefined
```

### 3. Reference Type：`.` 与 `()` 之间的"中间人"
**点 `.` 返回的不是函数，而是一个规范内部的 Reference Type 值**——三元组 `(base, name, strict)`：
- `base`：来源对象；
- `name`：属性名；
- `strict`：严格模式下为 `true`。

例如严格模式下访问 `user.hi`，实际得到的是：
```js
(user, "hi", true) // Reference Type 值（无法直接在代码里使用，仅存在于引擎内部）
```
- 当 `()` **直接**跟在点/方括号后面时，引擎从这个 Reference Type 里取出 `base`，把 `this` 设为 `user`，调用 `hi`。
- 任何**其他操作**（赋值、三元表达式、传参……）都会让 Reference Type "退壳"成普通的属性值（一个函数），来源对象信息随之丢弃 → 后续调用 `this` 丢失。

### 4. 结论与对策
- `this` 只在**直接**通过 `obj.method()` 或 `obj['method']()` 调用时才能正确传递。
- 动态拿到方法后想保住 `this`，常用手段：`func.bind(obj)`、箭头函数包裹 `() => user.hi()`、`obj.method.call(obj)`。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `obj.method()` 天然绑定 receiver，永远不可能"丢 this" | `.` 只取到函数值，`this` 依赖 Reference Type 传递 | Java 方法调用在字节码层面就带 `aload_0`（this），JS 的函数与对象是松耦合的 |
| 方法引用 `user::hi` **捕获**了 receiver | `user.hi.bind(user)` | Java 的方法引用自带绑定，等于 JS 手动 bind |
| 函数指针/委托概念（C# delegate 类比） | Reference Type `(base, name, strict)` | 都是"方法 + 目标对象"的组合体，JS 的只是规范内部类型 |

## 一句话总结
`obj.method()` 能拿到正确 `this`，是因为点运算符返回的不是函数本身，而是规范内部的 Reference Type `(base, name, strict)`，`()` 紧跟其后时从中取出 `base` 设为 `this`；一旦把方法赋值给变量、塞进三元表达式或以其他方式"中转"，Reference Type 就退化成裸函数、`this` 丢失——这就是 `(cond ? user.hi : user.bye)()` 报错的根因，修法是 `bind`、箭头函数包裹或 `call` 显式指定。
