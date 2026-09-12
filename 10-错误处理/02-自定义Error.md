# 02 自定义 Error，扩展 Error

> 原文：`zh.javascript.info/1-js/10-error-handling/2-custom-errors/article.md`
> 就是 Java 里自定义异常体系那套思路：`extends Error`、建层次结构、包装底层异常。差别在 `name` 属性要手动维护，以及 `cause` 在 ES2022 之前要自己塞。

## 知识点

### 1. 为什么要自定义 error

- 领域特定的失败需要自己的类型：`HttpError`（带 `statusCode`）、`DbError`、`NotFoundError`……
- 技术上 `throw` 可抛任何值，不必继承 `Error`；**但继承后才能用 `obj instanceof Error` 识别，且自动获得 `stack`，所以总是继承**。
- 应用变大后 error 自然形成层次结构：`HttpTimeoutError extends HttpError`。

### 2. 基本写法：`extends Error`

```js
class ValidationError extends Error {
  constructor(message) {
    super(message);              // (1) 必须：子类构造器要求先调 super，由父类设置 message
    this.name = "ValidationError"; // (2) super 会把 name 设成 "Error"，手动改回来
  }
}
```

使用时分流处理 + 不认识的就再抛出：

```js
try {
  readUser('{ "age": 25 }');
} catch (err) {
  if (err instanceof ValidationError) {
    alert("Invalid data: " + err.message);
  } else if (err instanceof SyntaxError) {
    alert("JSON Syntax Error: " + err.message);
  } else {
    throw err; // 未知 error，再抛
  }
}
```

- **`instanceof` 优于 `err.name == "SyntaxError"` 比较**：子类也能通过检查，面向未来（下面继承层次深了以后依然成立）。

### 3. 深入继承：携带额外信息

```js
class PropertyRequiredError extends ValidationError {
  constructor(property) {
    super("No property: " + property); // message 由构造器生成，调用方只传属性名
    this.name = "PropertyRequiredError";
    this.property = property;          // 附加领域信息
  }
}
```

### 4. 消除 `this.name = ...` 样板：自定义基础类

```js
class MyError extends Error {
  constructor(message) {
    super(message);
    this.name = this.constructor.name; // 自动等于子类名
  }
}

class ValidationError extends MyError { }
class PropertyRequiredError extends ValidationError {
  constructor(property) {
    super("No property: " + property);
    this.property = property;
  }
}

new PropertyRequiredError("field").name; // "PropertyRequiredError" ✔
```

### 5. 包装异常（wrapping exceptions）

场景：`readUser` 内部可能抛 `SyntaxError`、`ValidationError`、未来更多——调用方不想逐一 `instanceof`，只想知道"读数据失败了"。

做法（与 Java 的异常包装完全同构）：

1. 定义高层 error `ReadError` 表示抽象的"数据读取失败"；
2. 内部捕获底层 error，换成 `ReadError` 抛出；
3. 原始 error 存进 `cause` 属性，需要细节时还能查。

```js
class ReadError extends Error {
  constructor(message, cause) {
    super(message);
    this.cause = cause;
    this.name = 'ReadError';
  }
}

function readUser(json) {
  let user;
  try {
    user = JSON.parse(json);
  } catch (err) {
    if (err instanceof SyntaxError) throw new ReadError("Syntax Error", err);
    else throw err;
  }
  try {
    validateUser(user);
  } catch (err) {
    if (err instanceof ValidationError) throw new ReadError("Validation Error", err);
    else throw err;
  }
}

// 调用方只需检查一种类型
try {
  readUser('{bad json}');
} catch (e) {
  if (e instanceof ReadError) {
    alert("Original error: " + e.cause); // 细节还在
  } else {
    throw e;
  }
}
```

> 补充：ES2022 已把 `cause` 标准化，可直接 `new Error("msg", { cause: err })`，但教程原文是手动赋值 `this.cause`，老代码里两种都常见。

## 与 Java 对比

| Java | JS | 说明 |
|---|---|---|
| `class MyException extends RuntimeException` | `class MyError extends Error` | 同构；JS 无 checked/unchecked 之分，全 unchecked |
| 类名即类型名，`getClass().getSimpleName()` | `name` 是个普通字符串属性，`super()` 后默认是 `"Error"` | 要手动改；或用 `this.constructor.name` 基类技巧 |
| `new IOException("disk full", cause)`，JDK 内置 `getCause()` | 手动 `this.cause = cause`（ES2022 起 `new Error(msg, {cause})`） | "包装异常"思路完全一致，JS 旧代码靠自己约定属性名 |
| 多 catch 分支按类型分流 | 一个 `catch` 里 `if/else instanceof` | 参见上篇 |

## 一句话总结

自定义 error 就是 Java 异常体系那套：`extends Error`（记得 `super(message)` 并修 `name`）、按领域建层次、用 `instanceof` 分流、用"包装异常"（高层 error + `cause` 引用底层 error）向调用方屏蔽细节；与 Java 的差别仅是 `name` 要手动维护和 `cause` 曾非标准。
