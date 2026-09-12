# 07 Mixin 模式

> 原文：`zh.javascript.info/1-js/09-classes/07-mixins/article.md`
> JS 只有单继承（一个 `[[Prototype]]`、一个 `extends`），想给类"再加一组能力"就用 mixin：**把一个对象的方法拷贝到类的 prototype 上**。最接近 Java 接口的 default 方法，但能带状态。教程用一个可复用的事件系统 `eventMixin` 收尾。

## 知识点

### 1. 什么是 mixin
- 维基定义：一个类，其方法可被其他类使用，**而无需继承**。
- 换句话说：mixin 提供某种行为的实现，不单独使用，而是"混入"其他类。
- 场景：`User` 需要 `EventEmitter` 的能力；`StreetSweeper` + `Bicycle` → `StreetSweepingBicycle`。这类需求用单继承表达不了。

### 2. 最简实现：对象 + `Object.assign` 到 prototype
```js
let sayHiMixin = {
  sayHi()  { alert(`Hello ${this.name}`); },
  sayBye() { alert(`Bye ${this.name}`); }
};

class User {
  constructor(name) { this.name = name; }
}

Object.assign(User.prototype, sayHiMixin);   // 拷贝方法，不是继承
new User("Dude").sayHi();                    // Hello Dude
```
- **没有继承，只有方法拷贝**。`User.prototype` 上多了 `sayHi/sayBye` 两个自有属性，原型链不变。
- 因此可以和 `extends` 共存：`class User extends Person {...}` 之后再 `Object.assign(User.prototype, sayHiMixin)`，相当于"继承一个 + 混入多个"。
- 方法里的 `this` 照常是调用时点号前的对象（实例），所以 mixin 方法可以读写实例状态。
- 细节：`Object.assign` 拷过来的方法是**可枚举**的，与 class 自带方法（不可枚举）不同，`for..in` 会列出来；介意的话用 `Object.defineProperties(User.prototype, Object.getOwnPropertyDescriptors(mixin))` 拷描述符。

### 3. mixin 内部可以有自己的继承，`super` 走 mixin 的链
```js
let sayMixin = {
  say(phrase) { alert(phrase); }
};

let sayHiMixin = {
  __proto__: sayMixin,                           // 或 Object.setPrototypeOf
  sayHi()  { super.say(`Hello ${this.name}`); }, // (*)
  sayBye() { super.say(`Bye ${this.name}`); }
};

Object.assign(User.prototype, sayHiMixin);
new User("Dude").sayHi(); // Hello Dude
```
- `(*)` 处的 `super.say()` 在 **mixin 的原型链**（`sayHiMixin.[[Prototype]] = sayMixin`）中查找，**不是**在 `User` 的类层次里找。
- 原因是上一篇的 `[[HomeObject]]`：`sayHi` 定义在 `sayHiMixin` 里，即使被拷贝到 `User.prototype`，它的 `[[HomeObject]]` 仍是 `sayHiMixin`，`super` 解析为 `sayHiMixin.[[Prototype]].say`。
- 注意 `say` 本身并没有被拷到 `User.prototype`（`Object.assign` 只拷自有属性），但这不影响 `super.say()` 工作。

### 4. 实战：`eventMixin`，给任意类加事件能力
```js
let eventMixin = {
  // 订阅：menu.on('select', function(item) { ... })
  on(eventName, handler) {
    if (!this._eventHandlers) this._eventHandlers = {};        // 状态懒初始化，存在实例上
    if (!this._eventHandlers[eventName]) this._eventHandlers[eventName] = [];
    this._eventHandlers[eventName].push(handler);
  },

  // 取消订阅：menu.off('select', handler)
  off(eventName, handler) {
    let handlers = this._eventHandlers?.[eventName];
    if (!handlers) return;
    for (let i = 0; i < handlers.length; i++) {
      if (handlers[i] === handler) handlers.splice(i--, 1);    // 删掉后索引回退一位
    }
  },

  // 触发：this.trigger('select', data1, data2)
  trigger(eventName, ...args) {
    if (!this._eventHandlers?.[eventName]) return;             // 没人订阅
    this._eventHandlers[eventName].forEach(handler => handler.apply(this, args));
  }
};
```
用法：
```js
class Menu {
  choose(value) { this.trigger("select", value); }   // 业务代码只管触发
}
Object.assign(Menu.prototype, eventMixin);

let menu = new Menu();
menu.on("select", value => alert(`Value selected: ${value}`));
menu.choose("123"); // Value selected: 123
```
- `on/off/trigger` 三个方法构成一个最小的发布-订阅系统；处理器列表 `_eventHandlers` 放在**实例**上（`this._eventHandlers`），各实例互不干扰（回顾 08 章"状态放实例上"）。
- `handler.apply(this, args)`：处理器以触发者为 `this`、以 `trigger` 的额外参数为实参被调用。
- 用了 `?.` 可选链（04 章）和 rest 参数 `...args`（06 章）。
- 任何类都能 `Object.assign(X.prototype, eventMixin)` 获得事件能力，**不影响它原有的继承链**。Node 的 `EventEmitter`、浏览器的 `EventTarget` 就是这套思想的完整版。

### 5. 冲突风险
- mixin 是拷贝，**同名方法会静默覆盖**类里已有的方法（后 assign 的赢），不会报错。
- 所以 mixin 的方法名要仔细取，尽量降低撞名概率；多个 mixin 之间也要留意。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 单继承 + 可实现多个接口 | 单继承（一个 `[[Prototype]]`）+ 可混入多个 mixin | 解决同一类问题 |
| Java 8 接口 default 方法：提供行为，不能有实例字段 | mixin 对象：提供方法，且可通过 `this._xxx` 懒创建实例状态 | JS mixin 能带状态，Java default 方法不能（要状态得用抽象类或组合） |
| 两个接口 default 方法冲突 → **编译错误**，必须显式解决 | `Object.assign` 后者静默覆盖前者 | JS 冲突无提示，靠命名规避 |
| 接口是类型，`obj instanceof Iface` 可用 | mixin 不是类型，`instanceof` 无法判断"是否混入" | 需要判断能力时用 `"on" in obj` 之类的鸭子类型 |
| `PropertyChangeSupport` / 监听器接口 / `java.util.Observer` | `eventMixin` 的 `on/off/trigger` | JS 版几十行搞定，且可加到任何类上 |
| Lombok `@Delegate`、组合 + 转发 | `Object.assign(X.prototype, mixin)` 直接拷方法 | JS 的原型是可变对象，运行时即可增能力 |
| 接口 default 方法里 `Iface.super.m()` 调父接口 | mixin 里 `super.m()` 沿 mixin 自己的原型链找 | 两者都是"按定义处解析"，与调用者的类层次无关 |

## 一句话总结
mixin 是"一组可被其他类使用但不靠继承获得的方法"，JS 里最简做法是把方法写在一个普通对象里再 `Object.assign(Class.prototype, mixin)` 拷到原型上，与 `extends` 互不冲突、可混入多个；mixin 内部可以有自己的原型链，其中的 `super` 依 `[[HomeObject]]` 沿 **mixin 的**链解析而非目标类的链；`eventMixin` 用 `on/off/trigger` 三个方法把发布-订阅能力加到任意类上，处理器表懒初始化在实例的 `_eventHandlers` 上；由于是拷贝，同名方法会被静默覆盖，取名要小心。
