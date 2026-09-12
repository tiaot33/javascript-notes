# 08 async/await

> 原文：`zh.javascript.info/1-js/11-async/08-async-await/article.md`
> 对 Java 开发者最重要的一句话：**`await` 不阻塞线程**——它只是"暂停这个函数"，引擎同时去干别的活。语义像 C# 的 async/await；和 `future.get()` 的阻塞等待完全是两回事。

## 知识点

### 1. `async` 关键字：函数永远返回 promise

```js
async function f() {
  return 1;
}
f().then(alert); // 1 —— 返回值被自动包装成 resolved promise
```

- `async` 保证函数返回 promise：返回普通值自动包装为 resolved promise；显式返回 promise 也行，效果相同。
- class 方法同样适用：`class Waiter { async wait() { ... } }`。

### 2. `await`：暂停函数，等 promise settle

```js
async function f() {
  let promise = new Promise((resolve, reject) => {
    setTimeout(() => resolve("done!"), 1000)
  });

  let result = await promise; // 函数"暂停"在这，1 秒后带着结果继续

  alert(result); // "done!"
}
```

- 只在 `async` 函数内合法，否则 **SyntaxError**。
- 语义：等 promise settle，resolve → 返回结果；**不耗 CPU、不阻塞线程**，引擎可同时处理其他脚本和事件。
- `await` 后面跟的**不一定是 promise**：跟普通值（`await 1`）时等价于 `await Promise.resolve(1)`，直接拿该值继续——所以 `await` 一个可能为 promise 也可能为普通值的变量是安全的。
- 就是 `promise.then` 的更优雅写法，可读性大幅提升。

改写 promise 链示例（`showAvatar`）：

```js
async function showAvatar() {
  let response = await fetch('/article/promise-chaining/user.json');
  let user = await response.json();

  let githubResponse = await fetch(`https://api.github.com/users/${user.name}`);
  let githubUser = await githubResponse.json();

  let img = document.createElement('img');
  img.src = githubUser.avatar_url;
  document.body.append(img);

  await new Promise((resolve, reject) => setTimeout(resolve, 3000));

  img.remove();
  return githubUser;
}
```

`.then` 链变成了从上到下的"同步风格"代码。

### 3. 两个补充特性

- **顶层 `await`**：在现代浏览器的 **module** 中允许在顶层直接 `await`；非 module 环境的通用替代是包进匿名 async 函数：

```js
(async () => {
  let response = await fetch('/article/promise-chaining/user.json');
  let user = await response.json();
  ...
})();
```

- **`await` 接受 thenable**：和 `.then` 一样，任何带 `.then` 方法的对象都可 `await`（调用其 `then`，传入内建 `resolve/reject` 等待其一被调用）。

### 4. Error 处理：rejected promise → `throw`

`await promise` 遇到 rejection 时，**就像那一行写了 `throw` 一样抛出该 error**：

```js
async function f() {
  await Promise.reject(new Error("Whoops!"));
}
// 等价于
async function f() {
  throw new Error("Whoops!");
}
```

由此得到两种处理方式：

```js
// (a) 函数内 try..catch（常规同步语法！可包多行 await）
async function f() {
  try {
    let response = await fetch('/no-user-here');
    let user = await response.json();
  } catch(err) {
    alert(err); // fetch 和 response.json 的错都在这里捕获
  }
}

// (b) 不 try..catch：async 函数返回的 promise 变为 rejected，外面挂 .catch
async function f() {
  let response = await fetch('http://no-such-url');
}
f().catch(alert); // TypeError: failed to fetch
```

- (b) 忘了挂 `.catch` → 未处理的 promise error，用全局 `unhandledrejection` 兜底（见第 4 篇）。

### 5. 与 `.then/.catch` 和 `Promise.all` 的协作

- 用了 `async/await` 基本告别 `.then`：等待交给 `await`，错误交给 `try..catch`。
- 但在**代码顶层**（所有 async 函数之外）语法上不能用 `await`，仍需 `.then/.catch` 接收最终结果和漏出的 error。
- **并行等待多个 promise**：`await Promise.all([...])`，error 会从失败的 promise 经 `Promise.all` 正常传到 `try..catch`：

```js
let results = await Promise.all([
  fetch(url1),
  fetch(url2),
]);
```

## 与 Java 对比

| Java | JS | 说明 |
|---|---|---|
| `future.get()` **阻塞当前线程**等结果 | `await` **挂起函数**，线程去干别的活 | 本质差异；概念上更接近 Java 21 虚拟线程的廉价挂起，但 JS 是语言级语法 |
| 无对应语法（Project Loom 之前异步只能 `thenApply` 链） | `async/await` 语言级关键字 | 与 C# 的 async/await 同源的设计 |
| 方法返回 `CompletableFuture<T>` 全靠约定 | `async` 方法**保证**返回 promise，返回值自动包装 | JS 把约定变成语法 |
| — | `await` 普通值 ≡ `await Promise.resolve(v)`，直接继续 | 参数类型不用预先收窄成 promise |
| 异常从 future 传播到 `get()` 处用 `catch` 捕获 | rejection 在 `await` 处变 `throw`，普通 `try..catch` 即可 | 同步式的错误处理回归，不再要 `.exceptionally` |

## 一句话总结

`async` 让函数必返 promise 并解锁内部的 `await`；`await` 把函数挂起（不阻塞线程）直到 promise settle——resolve 则返回结果、reject 则原地 `throw`，于是异步代码能用同步的写法和普通的 `try..catch` 来组织；顶层作用域不能用 `await` 时退回 `.then/.catch`，并行等待用 `await Promise.all([...])`。
