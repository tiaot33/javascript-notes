# 06 类检查："instanceof"

> 原文：`zh.javascript.info/1-js/09-classes/06-instanceof/article.md`
> `instanceof` 长得和 Java 一样，但**它检查的是原型链，不是构造函数**，可以被改 `prototype` 骗过，也能用 `Symbol.hasInstance` 自定义。附赠一个比 `typeof` 强得多的类型探测技巧：`Object.prototype.toString.call(x)`。

## 知识点

### 1. `obj instanceof Class`
`obj` 是 `Class` 或其派生类的实例则返回 `true`，会考虑继承：

```js
class Rabbit {}
new Rabbit() instanceof Rabbit;   // true

function Rabbit2() {}             // 老式构造函数也行
new Rabbit2() instanceof Rabbit2; // true

let arr = [1, 2, 3];
arr instanceof Array;             // true
arr instanceof Object;            // true，Array 原型链上有 Object.prototype
```
用途之一：写**多态函数**，按参数类型分别处理。

### 2. 算法：先看 `Symbol.hasInstance`，再走原型链
1. 如果 `Class` 有静态方法 `Symbol.hasInstance`，直接调用它，结果即答案：

```js
class Animal {
  static [Symbol.hasInstance](obj) {   // 自定义：有 canEat 的都算 animal
    if (obj.canEat) return true;
  }
}
({ canEat: true }) instanceof Animal;  // true（鸭子类型式的 instanceof）
```
2. 绝大多数类没有它，此时标准逻辑是：**沿 `obj` 的原型链逐级比较，看 `Class.prototype` 是否出现在链上**：

```js
obj.__proto__ === Class.prototype?
obj.__proto__.__proto__ === Class.prototype?
obj.__proto__.__proto__.__proto__ === Class.prototype?
...   // 任一为 true 则返回 true；到链尾 null 仍未匹配则 false
```
继承场景在第二步匹配：`rabbit.__proto__ === Rabbit.prototype`（不等于 `Animal.prototype`）→ `rabbit.__proto__.__proto__ === Animal.prototype` ✓。

- 等价写法：`Class.prototype.isPrototypeOf(obj)`。`objA.isPrototypeOf(objB)` 判断 `objA` 是否在 `objB` 的原型链上。

### 3. 关键：构造函数本身不参与检查
检查只涉及**原型链**和 **`Class.prototype`**，所以：

```js
function Rabbit() {}
let rabbit = new Rabbit();
Rabbit.prototype = {};       // 事后换掉 prototype
rabbit instanceof Rabbit;    // false，"不再是 rabbit 了"
```
练习题反过来：两个毫不相关的函数共用同一个 `prototype`，实例就互相"属于"对方：

```js
function A() {}
function B() {}
A.prototype = B.prototype = {};
new A() instanceof B;        // true！a.__proto__ === B.prototype
```
结论：**在 `instanceof` 眼里，决定"类型"的是 `prototype` 对象，不是构造函数**。

### 4. 福利：`Object.prototype.toString.call(x)` 揭示类型
普通对象转字符串是 `[object Object]`，这个 `toString` 其实可以"借"来对任何值调用，结果随值的类型变化（教程简写为 `{}.toString.call(x)`，是同一个函数）：

```js
let s = Object.prototype.toString;

s.call([]);         // [object Array]
s.call(123);        // [object Number]
s.call(null);       // [object Null]
s.call(undefined);  // [object Undefined]
s.call(alert);      // [object Function]
s.call(new Date()); // [object Date]
```
- 用 `call` 把 `this` 指定为目标值（06 章 call/apply 的方法借用），内部算法检查 `this` 的类型并返回对应标签。
- 相当于**增强版 `typeof`**：`typeof` 对所有对象只会说 `"object"`，它却能区分 `Array/Date/Null/...`。

### 5. `Symbol.toStringTag`：自定义标签
```js
let user = { [Symbol.toStringTag]: "User" };
Object.prototype.toString.call(user);                  // [object User]

// 浏览器环境对象大多自带标签：
window[Symbol.toStringTag];                            // "Window"
Object.prototype.toString.call(new XMLHttpRequest());  // [object XMLHttpRequest]
```
- 输出就是 `[object <Symbol.toStringTag 的值>]`。
- 注意：自己写的 `class Rabbit {}` 实例默认没有标签，`String(rabbit)` 仍是 `[object Object]`；想拿类名用 `rabbit.constructor.name`（前提是 `constructor` 没被破坏），或在类里定义 `get [Symbol.toStringTag]() { return "Rabbit"; }`。

### 6. 三种类型检查手段对比
| | 用于 | 返回值 |
|---|---|---|
| `typeof` | 原始类型（对象一律 `"object"`，函数 `"function"`） | 字符串 |
| `{}.toString.call(x)` | 原始类型、内建对象、带 `Symbol.toStringTag` 的对象 | 字符串 `[object Xxx]` |
| `instanceof` | 对象，且要考虑类层次/继承 | `true/false` |

补充（教程未提）：`instanceof Array` 在跨 iframe/跨 realm 时会失效（不同全局环境的 `Array` 是不同的函数），判断数组请用 `Array.isArray(x)`，它跨 realm 也正确（ES5 之前的 polyfill 正是用 `toString` 标签实现的）。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `obj instanceof Foo` 编译期检查类型层次，运行时看类元数据 | 运行时沿原型链找 `Foo.prototype` | 语法相同，JS 的答案可以因 `prototype` 被替换而改变 |
| 类型不可伪造 | 共用 `prototype` 即"同类"；`Symbol.hasInstance` 可任意定义 | JS 的 `instanceof` 更像"结构/链检查"，而非身份检查 |
| 接口也能 `instanceof List` | 没有接口概念，只能对类/构造函数检查；鸭子类型用 `"method" in obj` 或 `Symbol.hasInstance` 模拟 | |
| `obj.getClass().getSimpleName()` | `obj.constructor.name` 或 `{}.toString.call(obj)` | 前者依赖 `constructor` 未被破坏，后者只对内建/带标签对象有意义 |
| `Object.toString()` 默认 `类名@哈希` | 默认 `[object Object]`，标签可用 `Symbol.toStringTag` 定制 | |
| 不同 ClassLoader 加载的同名类 `instanceof` 为 false | 不同 realm（iframe）的 `Array` 不同，`instanceof Array` 为 false → 用 `Array.isArray` | 同类问题，JS 更常见于浏览器多窗口 |
| `if (x instanceof Foo f)` 模式匹配 | 无 | |

## 一句话总结
`obj instanceof Class` 先看 `Class` 是否定义了静态 `Symbol.hasInstance`，没有则沿 `obj` 的原型链逐级比较是否等于 `Class.prototype`（等价于 `Class.prototype.isPrototypeOf(obj)`），构造函数本身不参与，因此事后替换 `prototype` 会让已有实例"不再属于"该类，两个函数共用 `prototype` 则实例互相属于对方；`typeof` 只能区分原始类型，`Object.prototype.toString.call(x)` 能对原始值、`null/undefined`、内建对象和带 `Symbol.toStringTag` 的对象返回 `[object Xxx]` 形式的精确标签，是增强版 `typeof`；涉及类层次和继承的检查用 `instanceof`，判断数组用 `Array.isArray`。
