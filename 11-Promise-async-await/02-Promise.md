# 02 Promise

> 原文：`zh.javascript.info/1-js/11-async/02-promise-basics/article.md`
> 对 Java 开发者最直接的类比是 **`CompletableFuture`**：`resolve/reject` ≈ `complete/completeExceptionally`，`.then/.catch/.finally` ≈ `whenComplete/exceptionally`。最大差异：executor 是**立即同步执行**的，不像 `supplyAsync` 那样提交到线程池。

## 知识点

### 1. 概念：连接"生产者"与"消费者"的对象

- **生产者代码**：做事要花时间（如网络加载）；
- **消费者代码**：想第一时间拿到成果，可以有多个；
- **Promise** 就是把两者连起来的特殊 JS 对象（类比"订阅列表"，但更强）。

### 2. 构造器与 executor

```js
let promise = new Promise(function(resolve, reject) {
  // executor（生产者代码），new Promise 时自动、立即运行
  setTimeout(() => resolve("done"), 1000);
});
```

- executor **自动且立即**执行（同步！不是异步调度）。
- `resolve(value)` / `reject(error)` 两个参数由 JS 引擎预定义，executor 完成后调其一：
  - `resolve(value)` —— 成功，结果为 `value`
  - `reject(error)` —— 出错，`error` 为 error 对象（惯例用 `Error` 及其子类）

### 3. 内部属性 state / result（外部不可直接访问）

| 属性 | 初始 | resolve(value) 后 | reject(error) 后 |
|---|---|---|---|
| `state` | `"pending"` | `"fulfilled"` | `"rejected"` |
| `result` | `undefined` | `value` | `error` |

- **settled**：resolved 或 rejected 的统称（与 pending 相对）。
- **只能 settle 一次**：之后再调 `resolve/reject` 一律被忽略；且只接收一个参数，多余的忽略。
- `resolve/reject` 也可以**立即**调用（如结果已缓存），不必非得异步。
- `reject` 的参数技术上**可以是任何类型**（和 `resolve` 一样），但**建议传 `Error` 或其子类对象**——理由和"`throw` 应抛 Error"相同：保留 `stack`，且下游 `.catch`/`instanceof` 才能正常工作。
- `state`/`result` 是内部的，只能通过 `.then/.catch/.finally` 间接观察。

### 4. 消费者：`.then` / `.catch`

```js
promise.then(
  result => { /* 成功时执行 */ },
  error  => { /* 失败时执行 */ }
);
```

- 只关心成功：`.then(f)` 传一个参数。
- 只关心失败：`.catch(f)`，完全等价于 `.then(null, f)`，只是简写。

### 5. 清理：`.finally`

```js
new Promise((resolve, reject) => { /* ... */ })
  .finally(() => stopLoadingIndicator) // settled 时必执行
  .then(result => show result, err => show error);
```

类似 `try..finally`，但与 `.then(f, f)` 有三个重要区别：

1. **处理程序没有参数**——不知道也不关心成败，只做常规清理；
2. **结果/error 会穿透 `finally` 传给下一个合适的处理程序**；
3. **返回值被忽略**；唯一例外：`finally` 里抛 error 时，该 error 取代之前的结果传下去。

### 6. 可随时订阅已 settled 的 promise

promise 已经 settled 后再 `.then`，处理程序依然会执行（异步地，见"微任务"篇）——比现实订阅列表强：不用在活动开始前注册。

### 7. 实战：promise 化 `loadScript`

```js
function loadScript(src) {
  return new Promise(function(resolve, reject) {
    let script = document.createElement('script');
    script.src = src;
    script.onload = () => resolve(script);
    script.onerror = () => reject(new Error(`Script load error for ${src}`));
    document.head.append(script);
  });
}

promise.then(
  script => alert(`${script.src} is loaded!`),
  error => alert(`Error: ${error.message}`)
);
promise.then(script => alert('Another handler...')); // 可多次订阅
```

**Promise vs 回调：**

| Promise | 回调 |
|---|---|
| 按自然顺序编码：先 `loadScript`，再 `.then` 处理结果 | 调用前就必须准备好回调函数 |
| 可在同一 promise 上多次 `.then`，任意多个订阅者 | 只能有一个回调 |

## 与 Java 对比

| Java（CompletableFuture） | JS（Promise） | 说明 |
|---|---|---|
| `supplyAsync(...)` 提交到线程池，惰性调度 | executor 在 `new Promise` 时**同步立即**执行 | Promise 本身不引入线程，只是"结果的容器 + 订阅机制" |
| `complete(v)` / `completeExceptionally(e)` | `resolve(v)` / `reject(e)` | 都只能生效一次 |
| `get()` 阻塞当前线程拿结果 | 无阻塞 API，只能 `.then` 订阅 | JS 主线程不允许阻塞 |
| `whenComplete` / `exceptionally` | `.then(f, f)` / `.catch` | `.catch(f)` ≡ `.then(null, f)` |
| `whenComplete` 拿得到结果/异常 | `.finally` 无参数、结果穿透 | 都用于清理；JS 的 finally 更"透明" |

## 一句话总结

Promise 是连接异步生产者与消费者的核心对象：`new Promise(executor)` 的 executor 同步立即执行，通过 `resolve/reject` 把内部状态从 pending 一次性钉死为 fulfilled/rejected；消费者用 `.then`（双参或单参）、`.catch`（≡`.then(null,f)`）、`.finally`（无参、穿透、清理专用）订阅结果，且订阅时机不限，settled 后补挂的处理程序照样执行。
