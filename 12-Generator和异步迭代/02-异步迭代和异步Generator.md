# 02 异步迭代和异步 Generator

> 原文：`zh.javascript.info/1-js/12-generators-iterators/2-async-iterators-generators/article.md`
> 同步迭代处理"值已经在那儿"的数据；当值要**异步到来**（`setTimeout`、网络分段下载）时，需要一套异步版迭代协议。Java 里没有直接的异步迭代协议，可以类比 `Flow.Publisher`（Reactive Streams）或手写"翻页循环"，JS 则用 `Symbol.asyncIterator` + `async function*` 把它做成了语言级语法。

## 知识点

### 1. 回顾：同步可迭代协议

- 对象想被 `for..of` 迭代，需实现 `Symbol.iterator` 方法：
  1. 循环开始时 `for..of` 调它一次，应返回一个带 `next()` 的对象；
  2. 每次迭代调用 `next()`；
  3. `next()` 返回 `{done: true/false, value}`，`done:true` 表示循环结束。

### 2. 异步可迭代对象：`Symbol.asyncIterator`

同步协议的三处替换，就能让对象支持"值异步到来"：

1. 用 `Symbol.asyncIterator` 取代 `Symbol.iterator`；
2. `next()` 返回一个 **Promise**（resolve 出 `{done, value}`）——直接写成 `async next()` 即可，还能在内部用 `await`；
3. 消费端用 `for await (let item of iterable)`（注意 `await` 关键字）。

```js
let range = {
  from: 1,
  to: 5,

  [Symbol.asyncIterator]() { // (1) for await..of 开始时调用一次
    return {
      current: this.from,
      last: this.to,

      async next() { // (2) 每次迭代调用，返回 promise
        await new Promise(resolve => setTimeout(resolve, 1000)); // (3) 内部可 await
        if (this.current <= this.last) {
          return { done: false, value: this.current++ };
        } else {
          return { done: true };
        }
      }
    };
  }
};

(async () => {
  for await (let value of range) { // (4) 每秒弹一个：1,2,3,4,5
    alert(value);
  }
})()
```

`for await..of` 的工作机制：调一次 `range[Symbol.asyncIterator]()` 拿到迭代器，然后反复 `await next()` 取值。

补充：`next()` 方法**可以不写成 `async`**，只要返回一个 promise 的普通方法也行；但写成 `async next()` 的好处是方法内部可以直接用 `await`（比如上面 1 秒的延迟）。

### 3. 同步 vs 异步 iterator 对比

|       | Iterator | 异步 iterator |
|-------|----------|---------------|
| 提供迭代器的方法 | `Symbol.iterator` | `Symbol.asyncIterator` |
| `next()` 返回 | 任意值（`{done, value}`） | `Promise`（resolve 出 `{done, value}`） |
| 循环语法 | `for..of` | `for await..of` |

### 4. 坑：spread 等同步工具不能用于异步迭代

需要常规同步 iterator 的功能，碰到异步可迭代对象全部失效：

```js
alert( [...range] ); // Error, no Symbol.iterator
```

spread 找的是 `Symbol.iterator`，不是 `Symbol.asyncIterator`；不带 `await` 的 `for..of` 同理。

### 5. 异步 generator：`async function*`

- 回顾：普通 generator（`function*`）可作 `Symbol.iterator` 的简写，但**内部不能 `await`**，值必须同步产出。
- 语法升级很简单——`function*` 前加 `async`：

```js
async function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) {
    await new Promise(resolve => setTimeout(resolve, 1000)); // 可以 await 了！
    yield i;
  }
}

(async () => {
  for await (let value of generateSequence(1, 5)) {
    alert(value); // 每秒一个：1,2,3,4,5
  }
})();
```

- 内部可以 `await`、依赖 promise、发网络请求。
- **引擎盖下的差异**：异步 generator 的 `next()` 是异步的，返回 promise：

```js
result = await generator.next(); // result = {value:…, done: true/false}
```

这正是它能与 `for await...of` 协作的原因。

### 6. 异步 generator 作 `Symbol.asyncIterator`

普通 generator 之于 `Symbol.iterator`，异步 generator 之于 `Symbol.asyncIterator`——同样是省代码的简写：

```js
let range = {
  from: 1,
  to: 5,

  async *[Symbol.asyncIterator]() { // 等价于 [Symbol.asyncIterator]: async function*() {
    for (let value = this.from; value <= this.to; value++) {
      await new Promise(resolve => setTimeout(resolve, 1000));
      yield value;
    }
  }
};
```

- 技术上可以**同时**给对象挂 `Symbol.iterator` 和 `Symbol.asyncIterator`（同步/异步都能迭代），但实际中这么做很奇怪。

### 7. 实战：用异步 generator 封装分页数据

场景：在线服务普遍分页返回数据。GitHub 的 commit 接口：

- 请求 `https://api.github.com/repos/<repo>/commits`；
- 返回 30 条 commit 的 JSON，响应的 **`Link` header** 里带下一页 URL；
- 拿着下一页 URL 继续请求，以此类推。

期望的消费方式——完全感知不到分页，只是一个 `for await..of`：

```js
for await (let commit of fetchCommits("username/repository")) {
  // 处理 commit
}
```

用异步 generator 实现：

```js
async function* fetchCommits(repo) {
  let url = `https://api.github.com/repos/${repo}/commits`;

  while (url) {
    const response = await fetch(url, {
      headers: {'User-Agent': 'Our script'}, // (1) GitHub 要求任意 User-Agent
    });

    const body = await response.json(); // (2) 一页 commit 的 JSON 数组

    // (3) 从 Link header 提取下一页 URL（格式特殊，用正则）
    let nextPage = response.headers.get('Link').match(/<(.*?)>; rel="next"/);
    nextPage = nextPage?.[1]; // 可选链：没有下一页时为 undefined

    url = nextPage;

    for (let commit of body) { // (4) 把这一页的 commit 逐个 yield
      yield commit;            //     全 yield 完才进入下一轮 while 拉下一页
    }
  }
}
```

要点：

- 惰性翻页：一页的数据 yield 完之前，不会发起下一个请求；
- 消费端可以随时 `break`（例如取够 100 条就停），不会再多请求；
- 从外部完全看不到分页机制，对调用方就是一个 commit 的异步流。

### 8. 总结与数据流

- 同步 iterator/generator：处理不耗时、现成的数据；异步版本 + `for await..of`：处理有延迟的数据。
- 两张速查表：

| 可迭代协议 | Iterable | 异步 Iterable |
|---|---|---|
| 提供迭代器的方法 | `Symbol.iterator` | `Symbol.asyncIterator` |
| `next()` 返回 | `{value:…, done: true/false}` | resolve 出 `{value:…, done: …}` 的 `Promise` |

| Generator | Generator | 异步 generator |
|---|---|---|
| 声明 | `function*` | `async function*` |
| `next()` 返回 | `{value:…, done: …}` | resolve 出 `{value:…, done: …}` 的 `Promise` |

- Web 开发常见**分段流动的数据流**（chunk-by-chunk）：下载/上传大文件等，都适合用异步 generator 处理。
- 延伸：浏览器环境还有专门的 **Streams API**，提供数据流转换、管道传递（边下载边转发到别处）等接口。

## 与 Java 对比

| Java | JS | 说明 |
|---|---|---|
| `Iterator<T>` 同步取值 | `Symbol.asyncIterator` + `next()` 返回 Promise | 异步版迭代协议，语言级支持 |
| 增强 for | `for await..of` | 同样是协议语法糖，多个 await |
| 无（Reactive Streams 的 `Flow.Publisher` 要手写一堆接口） | `async function*` 直接产出异步流 | JS 把发布者模式压缩成几行语法 |
| 分页手写 while 循环、页码状态外露 | async generator 把分页封装在流内 | 调用方只见数据流不见分页 |
| — | spread/`for..of` 等同步工具不认 `Symbol.asyncIterator` | 易踩坑，协议互不兼容 |

## 一句话总结

异步迭代 = `Symbol.asyncIterator`（`next()` 返回 Promise）+ `for await..of`；异步 generator（`async function*`）让产出异步值像写同步代码一样自然，也是实现 `Symbol.asyncIterator` 的最佳简写；典型实战是把"分页网络请求"封装成一个数据流，消费端无感翻页。
