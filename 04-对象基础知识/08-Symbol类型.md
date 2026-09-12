# 08 Symbol 类型

> 原文：`zh.javascript.info/1-js/04-object-basics/08-symbol/article.md`

## 知识点

### 0. 背景：属性键只有两种
对象属性键只能是**字符串**或 **symbol**；其他类型（数字、布尔）自动转字符串（`obj[1]` ≡ `obj["1"]`）。

### 1. symbol = 唯一标识符原始类型
```js
let id = Symbol();      // 创建
let id2 = Symbol("id"); // 可带描述（仅是标签，调试用）
```
- **保证唯一**：描述相同的两个 symbol 也不相等：

```js
Symbol("id") == Symbol("id"); // false
```
- 不要拿 Ruby 等其他语言的 symbol 类比，机制不同。

### 2. symbol 不会自动转字符串
- 大多数值都能隐式转字符串，symbol 例外——这是**语言保护**：

```js
let id = Symbol("id");
alert(id);            // TypeError！不能隐式转换
alert(id.toString()); // "Symbol(id)"，显式转换才行
alert(id.description); // "id"，只取描述
```

### 3. 主要用途一："隐藏"属性，避免命名冲突
- 想给"属于别人代码"的对象加属性，用 symbol 键**绝不会和别人的键冲突**：

```js
let user = { name: "John" }; // 第三方代码的对象

let id = Symbol("id");
user[id] = 1; // 第三方代码不知道这个 symbol，不会意外访问/覆盖
```
- 对比字符串键：`user.id = "Our id"` 之后别人再 `user.id = "Their id"` 就互相覆盖了。
- 对象字面量里用 symbol 键**必须加方括号**（否则键是字符串 `"id"`）：

```js
let user = {
  name: "John",
  [id]: 123   // 用变量 id 的值作为键
};
```

### 4. symbol 属性的"隐身"特性
- **`for..in` 跳过 symbol 属性**；`Object.keys()` 也忽略它们 → 别人的遍历不会碰到你的隐藏属性。
- 但 `Object.assign` **会同时复制字符串和 symbol 属性**（克隆时希望全拷，就是这么设计的）。
- 不是 100% 隐藏：`Object.getOwnPropertySymbols(obj)` 能拿到所有 symbol 键，`Reflect.ownKeys(obj)` 返回**全部**键。

### 5. 全局 symbol 注册表
- 想让"同名 symbol 是同一个"（跨代码共享），用注册表：

```js
let id = Symbol.for("id");      // 注册表里有则返回，没有则创建并存入
let idAgain = Symbol.for("id");
id === idAgain; // true
```
- 反向查询：`Symbol.keyFor(sym)` 由全局 symbol 取名字；**对非全局 symbol 返回 `undefined`**（普通 symbol 只能看 `.description`）。

### 6. 系统 symbol（well-known symbols）
JS 内建一堆 `Symbol.*` 用于微调对象行为，后续章节逐个遇到：
- `Symbol.iterator` —— 自定义迭代
- `Symbol.toPrimitive` —— 自定义对象→原始值转换
- `Symbol.hasInstance`、`Symbol.isConcatSpreadable` ……

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 无对应概念 | `Symbol()` 全局唯一值 | 最接近的是 `new Object()` 当 Map 的 key 用（身份唯一） |
| 字符串 intern / 枚举单例 | `Symbol.for(key)` 全局注册表 | 保证全应用拿到同一实例 |
| 包私有/私有字段防外部访问 | symbol 属性"半隐藏"：常规遍历和访问碰不到 | 不是真正的私有（`Object.getOwnPropertySymbols` 能拿到），真正私有靠 `#field`（class 章节） |
| 注解/接口标记能力（如 `Iterable`） | 系统 symbol（`Symbol.iterator` 等） | 对象实现了对应 symbol 方法就获得某种语言能力 |

## 一句话总结
Symbol 是唯一标识符原始类型：`Symbol()` 创建的值永远唯一，用作对象键可实现"防冲突的隐藏属性"（`for..in`/`Object.keys` 跳过）；`Symbol.for(key)` 提供全局注册表实现共享 symbol；语言内部用 `Symbol.iterator` 等系统 symbol 开放行为的自定义钩子。
