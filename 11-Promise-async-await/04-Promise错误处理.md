# 04 使用 promise 进行错误处理

> 原文：`zh.javascript.info/1-js/11-async/04-promise-error-handling/article.md`
> 链尾的 `.catch` ≈ 整条链外面包了一个 `try..catch`：executor 和每个处理程序都有"隐式 try..catch"保护，`throw` 自动变 reject，控制权跳到最近的 error 处理程序。

## 知识点

### 1. 链尾 `.catch` 捕获上方一切 error

```js
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
  // ...
  .catch(error => alert(error.message)); // 网络故障、JSON 非法……都会被这里捕获
```

- promise 被 reject 时，控制权移交**最近的** rejection 处理程序，`.catch` 不必紧跟出错的那一步。

### 2. 隐式 try..catch

executor 和**所有处理程序**都被"隐式 `try..catch`"包裹：抛异常 ≡ reject。

```js
new Promise((resolve, reject) => {
  throw new Error("Whoops!");  // 与 reject(new Error("Whoops!")) 完全等效
}).catch(alert);

new Promise((resolve, reject) => { resolve("ok"); })
  .then((result) => { blabla(); })  // 编程错误也一样：ReferenceError 被捕获
  .catch(alert);                    // ReferenceError: blabla is not defined
```

- 不只是 `throw`——**任何**异常（包括代码写错）都会转为 rejection，被最近的 `.catch` 接住。

### 3. 再次抛出（Rethrowing）

与同步 `try..catch` 同一规则：**`.catch` 只处理认识的 error，不认识的再 `throw`**。

```js
new Promise((resolve, reject) => {
  throw new Error("Whoops!");
}).catch(function(error) {
  if (error instanceof URIError) {
    // 处理它
  } else {
    alert("Can't handle such error");
    throw error; // 再抛 → 跳到下一个 .catch
  }
}).then(function() {
  /* 上个 catch 再抛了，这里不执行 */
}).catch(error => {
  alert(`The unknown error has occurred: ${error}`);
  // 正常结束 → 后续若有 .then 会继续走成功路径
});
```

- `.catch` 正常完成 → 执行流回到成功路径，下一个 `.then` 被调用；
- `.catch` 中 `throw` → 执行流跳到下一个 `.catch`。

### 4. 未处理的 rejection

链尾没有 `.catch` 时，error"卡住"无人处理：

```js
window.addEventListener('unhandledrejection', function(event) {
  alert(event.promise); // 产生该 error 的 promise
  alert(event.reason);  // 未处理的 error 对象
});

new Promise(function() { throw new Error("Whoops!"); }); // 没有 .catch
```

- JS 引擎跟踪未处理的 rejection，产生全局 error；浏览器用 **`unhandledrejection`** 事件（HTML 标准）捕获，`event.promise` / `event.reason` 两个专有属性。
- Node.js 等非浏览器环境有各自的机制。
- 此类 error 通常无法恢复：正确做法是**告知用户 + 上报服务器**。

### 5. 实践准则（原文总结）

1. `.catch` 处理一切 promise error：`reject()` 的、处理程序中 `throw` 的；`.then` 第二参数效果相同。
2. `.catch` 应放在"知道如何处理 error"的准确位置；分析 error（可借助自定义 error 类）并再抛未知的。
3. 无法恢复的地方可以不挂 `.catch`。
4. 任何情况都应有 `unhandledrejection` 兜底，跟踪漏网 error 并上报，让应用永不"无声死亡"。

## 与 Java 对比

| Java（CompletableFuture） | JS（Promise） | 说明 |
|---|---|---|
| 各 stage 内抛异常自动包装进 future | 处理程序的"隐式 try..catch"：`throw` ≡ reject | 机制同构：异常沿链传播 |
| `exceptionally` / `handle` | `.catch` | 处理后可返回值回到成功路径，等价 |
| `catch` 里 `throw` 继续上抛 | `.catch` 里 `throw` → 下一个 `.catch` | 与同步异常规则一致 |
| 不 `join()/get()` 异常就被**静默吞掉** | 引擎检测并触发 `unhandledrejection` 全局事件 | JS 比 CompletableFuture 更"吵"，漏处理会在控制台留痕 |

## 一句话总结

Promise 的错误处理就是"隐式 try..catch + 链式传播"：executor 和处理程序里抛的任何异常都变成 rejection，沿链跳到最近的 `.catch`（或 `.then` 第二参数）；`.catch` 里再 `throw` 则继续向后跳、正常返回则回到成功路径；没人接的 rejection 由 `unhandledrejection` 全局事件兜底上报。
