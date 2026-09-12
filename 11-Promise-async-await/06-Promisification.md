# 06 Promisification（Promise 化）

> 原文：`zh.javascript.info/1-js/11-async/06-promisify/article.md`
> 把"接受 error-first 回调的函数"机械地包装成"返回 promise 的函数"——就是给老式回调 API 手写适配器，类比 Java 里把监听器式 API 包成 `CompletableFuture`。

## 知识点

### 1. 定义

**Promisification**：将一个接受回调的函数，转换为返回 promise 的函数。大量库和函数基于回调，而 promise（尤其配 `async/await`）更好用，所以这种转换很日常。

### 2. 手工包装一例

```js
// 原始：loadScript(src, callback)，回调为 error-first 风格
let loadScriptPromise = function(src) {
  return new Promise((resolve, reject) => {
    loadScript(src, (err, script) => {   // 提供自己的回调做转换
      if (err) reject(err);
      else resolve(script);
    });
  });
};

// 用法：loadScriptPromise('path/script.js').then(...)
```

本质：新函数调用原函数，在自定义回调里把结果翻译成 `resolve/reject`。

### 3. 通用辅助函数 `promisify(f)`

```js
function promisify(f) {
  return function (...args) {            // 返回包装函数 (*)
    return new Promise((resolve, reject) => {
      function callback(err, result) {   // 自定义回调 (**)
        if (err) reject(err);
        else resolve(result);
      }
      args.push(callback);               // 把回调追加到参数末尾
      f.call(this, ...args);             // 转发调用，保留 this
    });
  };
}

let loadScriptPromise = promisify(loadScript);
```

要点：

- 假设原函数回调是 **`(err, result)` 双参 error-first** 形式（最常见的约定）；
- `f.call(this, ...args)` 保留调用上下文，方法调用也安全。

### 4. 多结果回调的升级版

原函数回调可能是 `callback(err, res1, res2, ...)`。加 `manyArgs` 开关：

```js
function promisify(f, manyArgs = false) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      function callback(err, ...results) {
        if (err) reject(err);
        else resolve(manyArgs ? results : results[0]); // 多参时以数组 resolve
      }
      args.push(callback);
      f.call(this, ...args);
    });
  };
}

// f = promisify(f, true);
// f(...).then(arrayOfResults => ..., err => ...);
```

### 5. 边界与现成工具

- **更奇特的回调格式**（如没有 `err`：`callback(result)`）只能手写包装，别套通用 helper。
- 现成方案：Node.js 内建 **`util.promisify`**；第三方如 `es6-promisify`。
- **重要限制**：promise 只能有一个结果，而回调技术上可被调用多次——**promisify 只适用于回调只被调一次的函数**（后续调用会被忽略）。

## 与 Java 对比

| Java | JS | 说明 |
|---|---|---|
| 把监听器/回调式 API 手工包成 `CompletableFuture`（`new CompletableFuture` + 回调里 `complete/completeExceptionally`） | `promisify(f)` 手写或用 `util.promisify` | 同一适配器模式；Node 把它做成了标准库函数 |
| Guava `ListenableFuture` → `CompletableFuture` 的各种桥接工具 | `es6-promisify` 等 npm 模块 | 生态需求完全一致 |
| — | 回调可被多次调用，promise 只能 settle 一次 | 事件流（多次触发）不适合 promisify，要用别的抽象（如 Observable） |

## 一句话总结

Promisification 就是给 error-first 回调函数套一层适配器：内部 `new Promise`，在自定义回调里 `err ? reject(err) : resolve(result)`；通用 `promisify(f)` 假设 `(err, result)` 签名、`f.call(this, ...args)` 保上下文，`manyArgs` 变体支持多结果回调；记住 promise 只能 settle 一次，所以只适用于单次触发的回调。
