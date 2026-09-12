# 01 错误处理，"try...catch"

> 原文：`zh.javascript.info/1-js/10-error-handling/1-try-catch/article.md`
> 语法和 Java 几乎一样，但有三个 Java 开发者必踩的差异：**没有 checked exception 的概念**、**`catch` 不能按类型分流（只能在块内 `instanceof`）**、**`try...catch` 跨不过异步边界**。

## 知识点

### 1. 基本语法与执行流程

```js
try {
  // 代码...
} catch (err) {
  // err 是 error 对象
}
```

- `try` 中出错 → 执行立即停止，跳到 `catch`；无错 → 跳过 `catch`。脚本不会"死亡"。

### 2. 两个硬性限制

- **只对运行时 error 有效**：语法错误（解析期错误）整个脚本都解析不了，`try...catch` 也救不了。
- **同步执行，跨不过异步边界**：包不住 `setTimeout` 等"计划执行"的代码——回调执行时引擎早已离开 `try...catch`：

```js
try {
  setTimeout(function() { noSuchVariable; }, 1000); // 1 秒后脚本死掉，catch 捕不到
} catch (err) { /* 不会执行 */ }

// 正确姿势：try...catch 放进异步回调内部
setTimeout(function() {
  try { noSuchVariable; } catch { alert("捕获了！"); }
}, 1000);
```

### 3. Error 对象的三个属性

- `name` —— error 名称（如 `"ReferenceError"`，即构造器名）
- `message` —— 文字描述
- `stack` —— 调用栈字符串（非标准但到处支持）

`alert(err)` 整体输出时是 `"name: message"` 形式。

### 4. 可选的 catch 绑定（ES2019）

不需要 error 详情时可省略参数：

```js
try { ... } catch { ... }  // 没有 (err)
```

### 5. `throw` 操作符

```js
throw new SyntaxError("数据不全：没有 name");
```

- 技术上 `throw` 可以抛**任何值**（数字、字符串都行），但惯例是抛带 `name`/`message` 的对象，最好继承 `Error`。
- 内建构造器：`Error`、`SyntaxError`、`ReferenceError`、`TypeError` 等；`name` 自动等于构造器名，`message` 来自参数。
- 典型用法：数据语法正确但语义不合法时（如 JSON 缺字段），自己 `throw`，让 `catch` 成为统一错误处理点。

### 6. 再次抛出（Rethrowing）——核心模式

**`catch` 只处理它认识的 error，其余一律再抛出去。**

```js
try {
  // ...
} catch (err) {
  if (err instanceof SyntaxError) {
    alert("JSON Error: " + err.message);
  } else {
    throw err; // 不认识，丢给外层
  }
}
```

- 用 `instanceof` 判断类型（也可看 `err.name` 或 `err.constructor.name`）。
- 抛出的 error 可被外层 `try...catch` 接住；没有外层则脚本死亡。
- 不分流全部吞掉是反模式：会把编程错误（如变量写错）误报成"数据错误"，难以调试。

### 7. `finally`

```js
try { ... } catch (err) { ... } finally { ... }
```

- **任何出口都执行**：正常结束、出错、甚至 `try` 里 `return`——**`finally` 会在控制权交给外部代码之前执行**（即先跑完 `finally`，`return` 才真正生效）。
- 用途：无论成败都要做的清理（停止计时、关闭连接、隐藏 loading）。
- **`try...finally`（无 catch）**也有用：不想就地处理 error，但要保证清理执行，error 继续向外抛。
- 注意：`let` 声明在 `try` 块内的变量，`finally`/`catch` 里不可见——需要跨块用的变量在 `try` 之前声明。

### 8. 全局 catch（环境相关，非语言核心）

- 浏览器：`window.onerror = function(message, url, line, col, error) {...}`
- Node.js：`process.on("uncaughtException")`
- 作用不是恢复执行（编程错误基本无法恢复），而是**记录/上报错误**（配合 error 日志服务）。

## 与 Java 对比

| Java | JS | 说明 |
|---|---|---|
| `try/catch/finally` | 同 | 语法、`finally` 在 `return` 前执行等语义完全一致 |
| checked exception，方法必须声明 `throws` | 无此概念 | 所有 error 都是"unchecked"的，编译器不强制捕获 |
| `catch (IOException \| SQLException e)` 按类型分流 | 单个 `catch (err)` + 块内 `if (err instanceof X)` | JS 没有 multi-catch，类型分流靠手写 `instanceof` |
| 只能 `throw` `Throwable` 子类 | `throw` 任何东西 | 但惯例抛 `Error` 子类，否则丢失 stack 等信息 |
| 线程内异常不跨线程传播 | `try...catch` 不跨异步回调 | `setTimeout`/Promise 里的错误要在回调内部自己 catch |
| `Thread.setDefaultUncaughtExceptionHandler` | `window.onerror` / `process.on("uncaughtException")` | 都是最后兜底，用于上报而非恢复 |

## 一句话总结

`try...catch...finally` 语法与 Java 一致，但 JS 的 error 全是 unchecked、`catch` 只有一个靠 `instanceof` 分流、`throw` 可抛任意值；特有的坑是 **`try...catch` 包不住异步回调**，异步代码要在回调内部自行捕获，漏网的 error 由 `window.onerror` 等全局钩子兜底。
