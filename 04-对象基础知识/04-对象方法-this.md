# 04 对象方法，"this"

> 原文：`zh.javascript.info/1-js/04-object-basics/04-object-methods/article.md`

## 知识点

### 1. 方法 = 存在对象属性里的函数
```js
let user = { name: "John" };

user.sayHi = function() { alert("Hello!"); }; // 函数表达式赋值
user.sayHi(); // Hello!
```
也可以先声明函数再赋给属性：`user.sayHi = sayHi;`

### 2. 方法简写（ES6）
```js
let user = {
  sayHi() {        // 与 sayHi: function() {...} 等价，推荐
    alert("Hello");
  }
};
```

### 3. 方法中的 `this` = 点之前的对象
```js
let user = {
  name: "John",
  sayHi() {
    alert(this.name); // this 就是调用时的 user
  }
};
user.sayHi(); // John
```
- **不要用外部变量名代替 `this`**：对象被复制/原变量被重写后会访问到错误的对象：

```js
let admin = user;
user = null;
admin.sayHi(); // 若内部写的是 user.name → TypeError；写 this.name 则正常
```

### 4. `this` 是"自由"的，运行时才确定 ⚠️ 本章重点
- JS 的 `this` **不绑定声明位置**，可用于任何函数，值在**调用时**才计算，取决于"点符号前"是谁：

```js
function sayHi() { alert(this.name); }

let user = { name: "John" };
let admin = { name: "Admin" };

user.f = sayHi;
admin.f = sayHi;

user.f();      // John（this == user）
admin.f();     // Admin（this == admin）
admin['f']();  // Admin（方括号调用一样算）
```
- **脱离对象直接调用**：严格模式下 `this === undefined`（访问属性会报错）；非严格模式下 `this` 是全局对象（浏览器里的 `window`，历史遗留，`"use strict"` 已修复）。
- 优点：函数可在不同对象间复用；缺点：更容易出错。

### 5. 箭头函数没有自己的 `this`
- 箭头函数里的 `this` **取自外部（包围它的普通函数/上下文）**：

```js
let user = {
  firstName: "Ilya",
  sayHi() {
    let arrow = () => alert(this.firstName); // this 来自 sayHi
    arrow();
  }
};
user.sayHi(); // Ilya
```
- 适合"不想要独立 this，只想用外层 this"的场景（回调里常用，后续章节深入）。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `this` 编译期绑定到所属实例 | `this` 运行时由**调用点**决定 | **最大差异**：同一函数挂到不同对象上，`this` 就不同 |
| 方法属于类，不能脱离类存在 | 函数是独立值，可自由赋给任何对象的属性 | 函数和方法之间没有静态绑定关系 |
| 没有"丢失 this"问题 | 方法被单独取出调用时 `this` 会丢（变 `undefined`/全局对象） | 把方法当回调传递是经典坑 |
| lambda 捕获外层变量 | 箭头函数连 `this` 也一起用外层的 | 回调里想要外层 `this`，用箭头函数 |
| 类方法写法 | 对象字面量方法简写 `sayHi() {}` | 语法相似，机制完全不同 |

**实践建议**：
- 规则一句话：`obj.f()` 调用期间 `this === obj`；没有点，`this` 就是 `undefined`（严格模式）。
- 看到函数里用了 `this`，先找"它被谁以点的方式调用"。

## 一句话总结
方法是挂在对象属性上的函数；`this` 在调用时动态确定为"点前面的对象"，与声明位置无关；脱离对象调用 `this` 为 `undefined`（严格模式）；箭头函数没有自己的 `this`，直接继承外层。
