# 08 调度：setTimeout 和 setInterval

> 原文：`zh.javascript.info/1-js/06-advanced-functions/08-settimeout-setinterval/article.md`
> 这两个方法不在 JS 规范内（属于环境 API），但所有浏览器和 Node.js 都支持。

## 知识点

### 1. 基本语法
```js
let timerId = setTimeout(func|code, [delay], [arg1], [arg2], ...);
let timerId = setInterval(func|code, [delay], [arg1], [arg2], ...);
```
- `setTimeout`：`delay` 毫秒后执行一次。
- `setInterval`：每 `delay` 毫秒循环执行。
- `arg1, arg2...` 会传给被执行的函数。
- ⚠️ **传函数引用，不要调用**（新手常犯错误）：

```js
setTimeout(sayHi(), 1000);  // ❌ 传入的是 sayHi() 的执行结果 undefined
setTimeout(sayHi, 1000);    // ✓
```

- 第一个参数可以是代码字符串，**不建议**，用箭头函数代替：
  `setTimeout(() => alert('Hello'), 1000)`。

### 2. 取消调度
```js
let timerId = setTimeout(...);
clearTimeout(timerId);
clearInterval(timerId);
```
- 浏览器中 `timerId` 是数字，Node.js 中是定时器对象，环境无统一规范。

### 3. 嵌套 setTimeout vs setInterval（重点）
两种周期性调度方式：

```js
// 方式一：setInterval
let timerId = setInterval(() => alert('tick'), 2000);

// 方式二：嵌套 setTimeout（更灵活、间隔更精确）
let timerId = setTimeout(function tick() {
  alert('tick');
  timerId = setTimeout(tick, 2000); // 在前一次执行完后才调度下一次
}, 2000);
```

区别：
- **`setInterval` 的实际调用间隔比设定的短**：函数执行时间"吃掉"间隔的一部分；若函数执行超过间隔，引擎立即再次执行，完全没有停顿。
- **嵌套 `setTimeout` 保证延时的精确**：下一次调用在上一次**执行完成时**才调度。
- 嵌套写法还能根据结果动态调整下一次的间隔（如服务器过载时任重倍退 `delay *= 2`）。

### 4. 垃圾回收
- 传入的函数被调度程序持有内部引用，不会被 GC 回收，直到 `clearTimeout/clearInterval`。
- 副作用：函数引用的外部变量（闭包）也随之存活，**不需要的定时器要及时清除**。

### 5. 零延时 setTimeout：`setTimeout(func, 0)`
- 让 `func` **尽快**执行，但要等当前脚本执行完毕后调度程序才会调用它：

```js
setTimeout(() => alert("World"));
alert("Hello"); // 先 "Hello" 后 "World"
```

- 浏览器的限制：**嵌套超过 5 层后，间隔被强制拉到至少 4ms**（HTML5 标准，历史遗留）。`setInterval` 同理。Node.js 无此限制，且有 `setImmediate` 可选。

### 6. 其他事实
- 所有调度方法都**不保证精确延时**：CPU 过载、后台页签、省电模式都会拖慢，最小分辨率可能劣化到 300~1000ms。
- 显示 `alert/confirm/prompt` 弹窗时浏览器内部定时器仍在走。
- 进阶技巧：用 `setTimeout(count, 0)` 把 CPU 密集任务切片，避免浏览器"挂起"，每轮之间让 UI 有机会响应/重绘。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `ScheduledExecutorService` | `setTimeout` / `setInterval` | `schedule` ≈ timeout，`scheduleAtFixedRate` ≈ interval |
| `scheduleAtFixedRate`（速率优先，间隔被吃） | `setInterval` | **完美对应**：都会让任务执行时间侵蚀间隔 |
| `scheduleWithFixedDelay`（前次结束后才开始计时） | 嵌套 `setTimeout` | **完美对应**：保证间隔精度，JS 嵌套写法同理 |
| `ScheduledFuture.cancel()` | `clearTimeout(timerId)` | 取消任务 |
| `Timer`（单线程老 API） | 浏览器单线程事件循环的定时器 | JS 天然单线程，定时回调都排队执行（参考事件循环） |

## 一句话总结
`setTimeout(f, ms)` 延迟执行一次、`setInterval(f, ms)` 周期执行，用 `clearTimeout/clearInterval(timerId)` 取消；传函数引用别带 `()`；周期任务优先用**嵌套 `setTimeout`**——它在前一次执行完后才调度下一次，间隔比 `setInterval` 精确且可动态调整；`setTimeout(f, 0)` 表示"当前脚本结束后尽快执行"（浏览器 5 层嵌套后强制 4ms 下限），定时器持有回调防 GC，不用就及时 clear。
