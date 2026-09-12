# 05 Promise API

> 原文：`zh.javascript.info/1-js/11-async/05-promise-api/article.md`
> `Promise` 类的 6 个静态方法，核心考点是四个组合器的**语义差异**（等全部/等首个成功/等首个定稿、出错怎么办）。实战最常用 `Promise.all`。

## 知识点

### 1. `Promise.all` —— 全要，一个失败全失败

```js
let promise = Promise.all(iterable); // 通常传 promise 数组
```

- **所有 promise 都 resolve 才 resolve**，结果是按**源数组顺序**排列的结果数组（与完成先后无关）：

```js
Promise.all([
  new Promise(resolve => setTimeout(() => resolve(1), 3000)),
  new Promise(resolve => setTimeout(() => resolve(2), 2000)),
  new Promise(resolve => setTimeout(() => resolve(3), 1000))
]).then(alert); // 1,2,3 —— 3 秒后，顺序仍按入参
```

- **任意一个 reject → 整体立即 reject**，error 就是第一个失败者的；其余 promise 的结果被忽略（它们仍会继续跑完，Promise 没有"取消"概念，取消要靠 `AbortController`）。
- 参数里混有**非 promise 值会原样进结果数组**（`Promise.all([p, 2, 3])` → `[r, 2, 3]`）。
- 常见套路：`urls.map(url => fetch(url))` 数据数组 → promise 数组 → `Promise.all` 并行：

```js
let requests = names.map(name => fetch(`https://api.github.com/users/${name}`));
Promise.all(requests)
  .then(responses => Promise.all(responses.map(r => r.json())))
  .then(users => users.forEach(user => alert(user.name)));
```

### 2. `Promise.allSettled`（ES2020）—— 全等到底，不论成败

```js
Promise.allSettled(urls.map(url => fetch(url)))
  .then(results => {
    // results: [
    //   {status: 'fulfilled', value: response},
    //   {status: 'rejected',  reason: errorObject}, ...
    // ]
  });
```

- 等所有 promise **settle**（不提前失败），返回`{status, value|reason}` 对象数组；
- 适用：想要尽可能多的结果，个别失败可容忍（如批量抓用户信息）。

### 3. `Promise.race` —— 第一个 settled 的赢（不论成败）

```js
Promise.race([
  new Promise(resolve => setTimeout(() => resolve(1), 1000)),
  new Promise((_, reject) => setTimeout(() => reject(new Error("Whoops!")), 2000)),
]).then(alert); // 1
```

- 第一个 settle 的 promise 的 result/error 成为整体结果，之后的全部被忽略。

### 4. `Promise.any`（ES2021）—— 第一个成功的赢

```js
Promise.any([
  new Promise((_, reject) => setTimeout(() => reject(new Error("Whoops!")), 1000)),
  new Promise(resolve => setTimeout(() => resolve(1), 2000)),
]).then(alert); // 1（第一个虽快但 rejected，跳过）
```

- 只等第一个 **fulfilled**；与 `race` 的差别就在对 rejection 的处理。
- **全部 rejected → 整体以 `AggregateError` reject**，所有 error 存在其 `errors` 数组属性里：

```js
Promise.any([...]).catch(error => {
  console.log(error.constructor.name); // AggregateError
  console.log(error.errors[0]);        // Error: Ouch!
});
```

### 5. `Promise.resolve` / `Promise.reject` —— 造已 settled 的 promise

- `Promise.resolve(value)` ≡ `new Promise(resolve => resolve(value))`；
- 典型用途：函数要返回 promise，但结果已缓存——直接包一层：

```js
function loadCached(url) {
  if (cache.has(url)) return Promise.resolve(cache.get(url)); // 保证返回值恒为 promise
  return fetch(url).then(...);
}
```

- `Promise.reject(error)` 几乎不用；现代代码里这两个方法多被 `async/await` 取代。

### 6. 速查表

| 方法 | 等到什么时候 | 结果 | 失败行为 |
|---|---|---|---|
| `Promise.all` | 全部 fulfilled | 结果数组（按入参序） | 任一 reject → 立即 reject |
| `Promise.allSettled` | 全部 settled | `{status, value/reason}` 数组 | 永不 reject |
| `Promise.race` | 首个 settled | 首个的 result 或 error | 首个是 reject 则 reject |
| `Promise.any` | 首个 fulfilled | 首个成功的 result | 全 reject → `AggregateError` |

## 与 Java 对比

| Java（CompletableFuture） | JS | 说明 |
|---|---|---|
| `CompletableFuture.allOf(...)` 返回 `CompletableFuture<Void>`，结果要自己再 `join` 一遍 | `Promise.all` 直接 resolve **结果数组** | JS 版更好用；Java 要手写收集 |
| `CompletableFuture.anyOf(...)` 首个完成（含异常完成） | `Promise.race` | 语义一致 |
| 无直接对应（要自己 `handle` 包装） | `Promise.allSettled` / `Promise.any` | ES2020/2021 补齐的组合器 |
| `completedFuture(v)` | `Promise.resolve(v)` | 相同用途 |
| 无 `AggregateError` | `Promise.any` 全灭时抛 `AggregateError`（`errors` 数组） | Java 没有聚合异常类型 |

## 一句话总结

四个组合器按"等待条件 × 失败策略"区分：`all` 全成才成、一败俱败；`allSettled` 不论成败全等到并给出 `{status,value|reason}` 清单；`race` 取首个定稿者（成败不论）；`any` 取首个成功者、全灭时抛 `AggregateError`；外加 `resolve/reject` 两个造 settled promise 的静态工厂——实战中 `Promise.all` 配合 `arr.map(fetch)` 的并行模式用得最多。
