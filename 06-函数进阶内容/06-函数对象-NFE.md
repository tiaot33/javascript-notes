# 06 函数对象，NFE

> 原文：`zh.javascript.info/1-js/06-advanced-functions/06-function-object/article.md`
> 函数的类型是**对象**——可以调用，也可以增删属性、按引用传递。

## 知识点

### 1. 属性 `name`
- 返回函数名。**"上下文命名"**很智能：定义时没写名字，也会从上下文推测：

```js
function sayHi() {}
sayHi.name; // "sayHi"

let sayHi = function() {};
sayHi.name; // "sayHi"（从变量名推测）

function f(cb = function() {}) { alert(cb.name); }
f(); // "cb"（从默认参数名推测）

let user = { sayHi() {}, sayBye: function() {} };
user.sayHi.name; // "sayHi"
user.sayBye.name; // "sayBye"

[function() {}][0].name; // ""（数组里推测不出来）
```

### 2. 属性 `length`
- 返回**声明的入参个数**，**rest 参数不计数**：

```js
function f1(a) {}
function many(a, b, ...more) {}
f1.length;   // 1
many.length; // 2
```

- 实战用途：**内省（introspection）**——根据 handler 的参数个数做不同处理（多态）：

```js
function ask(question, ...handlers) {
  let isYes = confirm(question);
  for (let handler of handlers) {
    if (handler.length == 0) {
      if (isYes) handler();      // 无参 handler：只在肯定时调用
    } else {
      handler(isYes);            // 有参 handler：两种情况都调用
    }
  }
}
ask("Question?", () => alert('You said yes'), result => alert(result));
```

### 3. 自定义属性
函数是对象，可以挂自己的属性：

```js
function sayHi() {
  sayHi.counter++; // 统计调用次数
}
sayHi.counter = 0;
```

- ⚠️ **函数属性 ≠ 局部变量**：`sayHi.counter` 和函数内的 `let counter` 毫不相关，两条平行线。
- 可以**替代闭包**存状态，区别在可见性：

```js
function makeCounter() {
  function counter() { return counter.count++; }
  counter.count = 0;
  return counter;
}
let counter = makeCounter();
counter.count = 10; // ← 外部可访问可改！闭包版则完全私有
```
- 要**私有状态**用闭包，要**暴露可配置状态**用函数属性。jQuery 的 `$`、lodash 的 `_.add` 就是"主函数 + 挂载属性"，只污染一个全局名。

### 4. 命名函数表达式（NFE）
给函数表达式加一个内部名字：

```js
let sayHi = function func(who) {
  if (who) {
    alert(`Hello, ${who}`);
  } else {
    func("Guest"); // 内部用 func 自引用
  }
};
sayHi();  // Hello, Guest
func();   // ❌ Error: func is not defined —— 外部不可见
```

**名字的两个特性**：
1. 允许函数在**内部可靠地引用自己**（递归）。
2. 在**函数外不可见**。

**为什么不用外部变量名自调用？** 外部变量可能被改：

```js
let sayHi = function(who) {
  if (!who) sayHi("Guest"); // ⚠️ 依赖外部变量 sayHi
};
let welcome = sayHi;
sayHi = null;
welcome(); // ❌ Error: sayHi is not a function
// 用 NFE 的 func 则完全不受影响
```

- ⚠️ 这个"内部名"特性**只有函数表达式有**，函数声明没有此语法。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 方法不是对象，不能挂属性 | 函数是对象，可增删属性 | JS 独有；`sayHi.counter = 0` 这种写法 Java 没有对应 |
| `Method.getName()`（反射） | `func.name` | JS 直接是属性，无需反射 |
| `Method.getParameterCount()`（反射） | `func.length` | rest 参数不计数；用参数个数做重载式分发在 JS 库里常见 |
| 匿名内部类里用 `Outer.this` 自引用 | NFE 内部名 `func` | 都解决"匿名实体如何引用自己"的问题 |
| 枚举/工具类的静态常量挂状态 | 函数属性挂状态 | 闭包私有 ≈ `private` 字段，函数属性 ≈ `public` 字段 |

## 一句话总结
函数是对象：`name` 属性返回函数名（可上下文推测），`length` 返回声明的入参数（rest 不计），还能挂**自定义属性**替代闭包存状态（区别：属性公开、闭包私有）；**NFE**（`let f = function func() {...}`）给函数表达式一个仅内部可见、不受外部变量变化影响的自引用名，是函数表达式里做可靠递归的正确姿势。
