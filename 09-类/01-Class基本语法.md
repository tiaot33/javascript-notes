# 01 Class 基本语法

> 原文：`zh.javascript.info/1-js/09-classes/01-class/article.md`
> ES6 新增的 `class` 语法。对 Java 开发者来说最"眼熟"的一章，但**它本质上是上一章"构造函数 + prototype"的语法糖再加几条特殊规则**，理解这一点才不会被表象骗到。

## 知识点

### 1. 基本语法
```js
class User {
  constructor(name) {   // new 时自动调用
    this.name = name;
  }
  sayHi() {             // 方法之间没有逗号！
    alert(this.name);
  }
}
let user = new User("John");
user.sayHi();
```
- `new User("John")`：创建新对象 → 以参数运行 `constructor` → `this` 指向新对象。
- 没写 `constructor` 就视为空的 `constructor() {}`。
- **方法之间不能加逗号**，那是对象字面量的写法，放到类里是语法错误。

### 2. class 到底是什么：一种函数
```js
typeof User;                                // "function"
User === User.prototype.constructor;        // true
Object.getOwnPropertyNames(User.prototype); // ["constructor", "sayHi"]
```
`class User {...}` 实际做了两件事：
1. 创建一个名为 `User` 的函数，函数体来自 `constructor`。
2. 把类里的方法（`sayHi` 等）放进 `User.prototype`。

所以 `new User` 出来的对象通过 `[[Prototype]]` 拿到方法，机制与上一章 `F.prototype` 完全相同。等价的 ES5 写法：

```js
function User(name) { this.name = name; }
User.prototype.sayHi = function() { alert(this.name); };
```

### 3. 不仅仅是语法糖：三条特殊规则
| 差异 | 说明 |
|---|---|
| 内部标记 `[[IsClassConstructor]]: true` | **必须用 `new` 调用**，直接 `User()` 抛 `TypeError: Class constructor User cannot be invoked without 'new'`；`String(User)` 以 `"class User..."` 开头 |
| 类方法不可枚举 | `prototype` 上所有方法 `enumerable: false`，`for..in` 遍历实例时不会把方法列出来 |
| 类体自动严格模式 | 类里的所有代码都在 `"use strict"` 下运行，不必手写 |

补充（教程未提）：类声明**不像函数声明那样提升**，在声明之前 `new User()` 会抛 `ReferenceError`（暂时性死区，同 `let`）。

### 4. 类表达式
类和函数一样是"一等公民"，可以赋值、传参、返回：

```js
let User = class {            // 类表达式
  sayHi() { alert("Hello"); }
};

let User = class MyClass {    // 命名类表达式：MyClass 只在类体内可见
  sayHi() { alert(MyClass); }
};
alert(MyClass);               // Error，外部不可见

function makeClass(phrase) {  // 按需动态造类
  return class {
    sayHi() { alert(phrase); }
  };
}
let User = makeClass("Hello");
new User().sayHi();           // Hello
```

### 5. getter/setter 与计算属性名
与对象字面量一样，类里可以写 `get/set` 和 `[...]` 计算方法名，它们实际被定义在 `User.prototype` 上：

```js
class User {
  constructor(name) { this.name = name; } // 这一句会走 setter
  get name() { return this._name; }
  set name(value) {
    if (value.length < 4) { alert("Name is too short."); return; }
    this._name = value;
  }
  ['say' + 'Hi']() { alert("Hello"); }    // 计算方法名 → sayHi
}
```

### 6. 类字段（class fields）
```js
class User {
  name = "John";                          // 类字段，右边可以是任意表达式
  age = prompt("Age?", 18);               // 甚至函数调用
  sayHi() { alert(`Hello, ${this.name}!`); }
}
let user = new User();
user.name;            // "John"
User.prototype.name;  // undefined ← 字段在实例上，不在 prototype 上
```
- 语法 `<property name> = <value>`，写在类体里、方法之外。
- **字段挂在每个实例对象上**，方法挂在 `prototype` 上，这是二者的本质区别（对照上一章"状态放实例、方法放原型"）。
- 较新的特性（ES2022 定稿），老浏览器需要 polyfill/转译。

### 7. 用类字段 + 箭头函数做"绑定方法"
经典的"丢失 `this`"问题：方法作为回调传出去后 `this` 不再是对象。

```js
class Button {
  constructor(value) { this.value = value; }
  click() { alert(this.value); }
}
let button = new Button("hello");
setTimeout(button.click, 1000);   // undefined，this 丢了
```
06 章讲过两种修法：包装 `() => button.click()`，或在 constructor 里 `this.click = this.click.bind(this)`。类字段提供了第三种、最优雅的写法：

```js
class Button {
  constructor(value) { this.value = value; }
  click = () => {                 // 类字段 + 箭头函数
    alert(this.value);
  };
}
setTimeout(button.click, 1000);   // hello
```
- 箭头函数没有自己的 `this`，取定义时外层的 `this`，而类字段初始化时 `this` 就是正在创建的实例。
- 代价：`click` 是**每个实例一份**的函数（放在实例上，不共享），不在 `prototype` 上，子类也无法用 `super.click()` 调它。适合事件处理器这种要传出去的方法，普通方法仍应写成原型方法。

### 8. 练习题要点：把函数式 `Clock` 改写成 class
- 原版用闭包：`let timer` 和 `function render()` 是构造函数内部的局部变量，天然私有。
- class 版只能写成 `this.timer` 和 `render()` 方法，它们变成了**公开成员**，这是 class 语法（在私有字段 `#` 出现前）相对闭包写法的一个损失。
- `setInterval(() => this.render(), 1000)` 必须用箭头函数包一层，直接传 `this.render` 会丢 `this`。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `class` 是编译期类型，不是值 | `class` 是函数，是运行时的一等值 | 可赋值给变量、当参数传、从函数返回、动态生成，Java 只能靠反射或匿名类勉强模拟 |
| 类定义就是模板，方法在方法区 | 类 = 构造函数 + `prototype` 对象 | `class` 只是把上一章的写法包装了一下，`User.prototype.sayHi` 依然在 |
| 构造器不加 `new` 无法调用（语法层面） | 不加 `new` 抛 `TypeError`（运行时） | ES5 构造函数不加 `new` 时（非严格模式）会把属性静默写到全局对象上，class 修正了这个坑 |
| 字段声明 `String name = "John";` | 类字段 `name = "John";` | 语义相近：每个实例一份。但 JS 字段是普通可枚举属性，没有类型 |
| 方法引用 `button::click` 自带 receiver | `button.click` 传出去丢 `this`，需 `click = () => {...}` | Java 开发者最容易踩的坑，见第 7 点 |
| 方法重载（同名不同参数） | 无重载，同名方法后者覆盖前者 | 用默认参数、rest 参数或类型判断替代 |
| `getName()/setName()` 方法 | `get name()/set name()` 访问器，用起来像字段 | 同 07 章 |
| 成员之间用 `;` 分隔 | 方法之间无分隔符，字段以 `;` 结束 | 别在方法之间写逗号 |
| 类声明对整个编译单元可见 | 类声明在 TDZ 中，声明前不可用 | 与 `let/const` 行为一致 |

## 一句话总结
`class` 是"构造函数 + prototype 上的方法"的语法糖：`typeof User` 是 `"function"`，方法在 `User.prototype` 上，`constructor` 就是那个函数；但它多了三条硬规则：必须 `new` 调用、方法不可枚举、类体自动严格模式。类可以当表达式赋值/传递/动态生成，支持 `get/set` 和计算方法名；类字段 `name = value` 挂在**实例**上而非原型，用 `click = () => {...}` 形式的类字段可以做出永不丢 `this` 的绑定方法，代价是每个实例各持一份。
