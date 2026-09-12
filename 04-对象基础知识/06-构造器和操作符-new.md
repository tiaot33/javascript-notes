# 06 构造器和操作符 "new"

> 原文：`zh.javascript.info/1-js/04-object-basics/06-constructor-new/article.md`

## 知识点

### 1. 构造函数 = 普通函数 + 两条约定
用于批量创建相似对象。两条约定：
1. **命名首字母大写**。
2. **只能用 `new` 调用**。

```js
function User(name) {
  this.name = name;
  this.isAdmin = false;
}

let user = new User("Jack");
user.name;    // Jack
user.isAdmin; // false
```

### 2. `new` 调用时发生了什么（三步）
1. 创建一个新的空对象，赋给 `this`。
2. 执行函数体（通常给 `this` 加属性）。
3. **隐式返回 `this`**。

```js
function User(name) {
  // this = {};（隐式创建）
  this.name = name;
  this.isAdmin = false;
  // return this;（隐式返回）
}
```
- 技术上**任何函数（除箭头函数）都能当构造器**，"首字母大写"只是约定，用来提示"请用 new 调用"。

### 3. `new.target`（进阶，用得少）
- 函数内部用 `new.target` 检测是否被 `new` 调用：
  - 普通调用：`new.target === undefined`
  - `new` 调用：`new.target ===` 该函数本身
- 可借此让函数"无论是否加 new 都正常"（库里偶尔用，日常不推荐，会掩盖"正在创建对象"这一事实）。

### 4. 构造器的 `return` 规则
构造器通常不写 `return`。如果写了：
- `return` 一个**对象** → 返回这个对象，`this` 被丢弃。
- `return` 原始值或不写 → 忽略，返回 `this`。

```js
function BigUser() {
  this.name = "John";
  return { name: "Godzilla" };
}
new BigUser().name; // Godzilla

function SmallUser() {
  this.name = "John";
  return; // 等价于没写
}
new SmallUser().name; // John
```
- 无参数时可省略括号 `new User`（规范允许，但不是好风格）。

### 5. 构造器里也可以加方法
```js
function User(name) {
  this.name = name;
  this.sayHi = function() {
    alert("My name is: " + this.name);
  };
}
new User("John").sayHi(); // My name is: John
```
- 更高级的创建语法是 ES6 的 `class`（后续章节讲），构造器是其底层机制。

### 6. 小技巧：`new function() { ... }`
创建单个复杂对象时，可定义匿名构造器并立即 `new` 调用——只调用一次、无法复用，纯粹为了封装创建逻辑。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 构造器是类里的特殊方法，必须依附类 | 构造器是**普通函数**，约定大写开头 | JS 不需要类也能"构造" |
| `new User("x")` | `new User("x")` | 语法相同，`new` 的语义都是"建空对象→初始化→返回" |
| 构造器不能返回对象 | 构造器 `return` 对象会**覆盖**默认的 `this` | JS 特有的怪规则，知道即可，别用 |
| 缺失构造器编译报错 | 任何函数都能 `new`，错了运行时才发现 | 靠约定（大写）和 review 约束 |
| 类是唯一的对象模板 | 构造器/`class`/字面量/工厂函数都可以 | JS 建对象的路子很多，`class` 只是其中一种糖 |

## 一句话总结
构造器就是约定"大写开头 + 用 `new` 调用"的普通函数：`new` 负责创建空对象给 `this`、执行函数体、返回 `this`；`return` 对象会覆盖结果，返回原始值则被忽略；ES6 的 `class` 是建立在这套机制之上的语法糖。
