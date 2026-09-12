# 03 Promise 链

> 原文：`zh.javascript.info/1-js/11-async/03-promise-chaining/article.md`
> 关键机制一句话：**每个 `.then` 都返回新 promise，处理程序的返回值（普通值或 promise）决定下一个 `.then` 拿到什么**。这就是 Java 里 `thenApply` 与 `thenCompose` 的区别被 JS 合并进同一个 `.then` 的原因。

## 知识点

### 1. 链式传递的原理

```js
new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
}).then(function(result) {
  alert(result); // 1
  return result * 2;
}).then(function(result) {
  alert(result); // 2
  return result * 2;
}).then(function(result) {
  alert(result); // 4
});
```

- 每次 `.then` 调用都**返回一个新 promise**，所以能在其上继续 `.then`。
- 处理程序**返回的值**成为新 promise 的 result，传给下一个 `.then`。

### 2. 新手经典错误：多个 `.then` ≠ 链

```js
promise.then(result => { alert(result); return result * 2; }); // alert 1
promise.then(result => { alert(result); return result * 2; }); // alert 1（不是 2！）
promise.then(result => { alert(result); });                    // alert 1
```

- 挂在**同一个 promise** 上的多个 `.then` 彼此独立，拿到的都是**该 promise 自己的结果**，互不传值。
- 实际中极少这么用；要串联就用链。

### 3. 处理程序返回 promise → 链等待它 settled

```js
new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
}).then(function(result) {
  alert(result); // 1
  return new Promise((resolve, reject) => { // (*)
    setTimeout(() => resolve(result * 2), 1000);
  });
}).then(function(result) { // 等 (*) settle 后才执行
  alert(result); // 2
});
```

- 返回 promise 使链的其余部分**等待它 settled**，再用其结果继续 → 可构建**异步行为链**。
- 顺序加载脚本、链依然是"扁平"的（向下长而不是向右），告别金字塔：

```js
loadScript("/article/promise-chaining/one.js")
  .then(script => loadScript("/article/promise-chaining/two.js"))
  .then(script => loadScript("/article/promise-chaining/three.js"))
  .then(script => { one(); two(); three(); });
```

- 嵌套 `.then` 也能做到（且内层能访问外层的 `script1/script2/script3` 闭包变量），但会向右长——那是例外不是规则，**链式是首选**。

### 4. Thenable：鸭子类型的 promise

- 处理程序返回的可以是任意带 `.then(resolve, reject)` 方法的对象（"thenable"），JS 会当 promise 对待：调用它的 `then`，传入原生 `resolve/reject`，等其一被调用后沿链传递。
- 意义：第三方库可实现自己的 promise 兼容对象，与原生链无缝集成，**无需继承 `Promise`**。

### 5. 复杂示例：fetch

```js
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())  // response.json() 返回解析 JSON 的 promise
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
  .then(githubUser => {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    document.body.append(img);
    setTimeout(() => img.remove(), 3000); // (*) 问题：3 秒后链无法继续
  });
```

- `fetch(url)` 返回 promise，服务器返回 header 时以 `response` 对象 resolve（不必等 body 下完）；
- `response.text()` / `response.json()` 返回以完整内容 resolve 的 promise。

**可扩展性要点**：`(*)` 处 `setTimeout` 之后想再接步骤就接不上。修法是让处理程序返回 promise：

```js
.then(githubUser => new Promise(function(resolve, reject) {
  let img = document.createElement('img');
  img.src = githubUser.avatar_url;
  document.body.append(img);
  setTimeout(() => {
    img.remove();
    resolve(githubUser); // 3 秒后链继续
  }, 3000);
}))
.then(githubUser => alert(`Finished showing ${githubUser.name}`));
```

### 6. 最佳实践

- **异步行为应始终返回 promise**：即使现在不打算扩展链，以后可能要。
- 把各步拆成可复用函数（`loadJson` / `loadGithubUser` / `showAvatar`），主链变成一张"流程说明书"。

## 与 Java 对比

| Java（CompletableFuture） | JS（Promise） | 说明 |
|---|---|---|
| `thenApply(f)`：f 返回普通值 | `.then` 返回普通值 | 同：值直接成为下一个阶段的结果 |
| `thenCompose(f)`：f 返回另一个 Future，摊平嵌套 | `.then` 返回 promise，**自动摊平** | JS 动态类型把 map/flatMap 合并进一个 `.then`；Java 静态类型必须分两个方法 |
| `CompletionStage` 接口 | thenable（鸭子类型） | 有 `.then` 方法就算数，连接口/继承都不要 |
| 链式写法 `future.thenApply(...).thenCompose(...)` | `.then(...).then(...)` | 都是把"回调金字塔"拉平成纵向链 |

## 一句话总结

Promise 链的核心规则只有一条：`.then` 返回新 promise，处理程序返回普通值则直接传递、返回 promise（或 thenable）则后续处理程序等它 settled 再拿其结果——借此把异步步骤纵向串成扁平链；注意别把"同一 promise 挂多个 `.then`"误当成链，并让每个异步函数都返回 promise 以保持链可扩展。
