# 01 Generator

> 原文：`zh.javascript.info/1-js/12-generators-iterators/1-generators/article.md`
> 普通函数一次 `return` 到底；generator 是**能"暂停/恢复"的函数**：逐个 `yield` 值出去，外部还能把值塞回来。Java 没有对应物，最接近的是手写一个保存状态的 `Iterator`（状态机自己维护），或者 Kotlin/Python 的协程式 generator。

## 知识点

### 1. generator 函数：`function*`

- 语法：`function* f(…) {…}`，内部用 `yield` 产出一个接一个的值：

```js
function* generateSequence() {
  yield 1;
  yield 2;
  return 3;
}
```

- `function* f(…)` 和 `function *f(…)` 两种写法都对，习惯用前者：`*` 描述的是"函数种类"而非名字，紧贴 `function`。

### 2. 调用 generator 函数 ≠ 执行函数体

- 调用 `generateSequence()` 时函数体代码**完全不执行**，而是返回一个 **generator 对象**，由它管理执行流程：

```js
let generator = generateSequence(); // 函数体一行都没跑
```

### 3. `next()`：推进到下一个 yield

- 每次调用 `generator.next()`：恢复执行 → 运行到最近的 `yield <value>`（value 可省略，默认 `undefined`）→ **暂停**，把值交回外部。
- `next()` 的返回值固定是两个属性的对象（可用 `JSON.stringify(one)` 直接查看）：
  - `value`：yield 产出的值；
  - `done`：函数是否已执行完（`true`/`false`）。

```js
generator.next(); // {value: 1, done: false}
generator.next(); // {value: 2, done: false}
generator.next(); // {value: 3, done: true}  —— 执行到 return 3
generator.next(); // {done: true}            —— 完成后再调无意义，永远这个结果
```

### 4. generator 是可迭代的

- 有 `next()` 就意味着满足 iterable 协议，可直接 `for..of`：

```js
for (let value of generateSequence()) {
  alert(value); // 1，然后 2 —— 注意没有 3！
}
```

- **坑：`for..of` 会忽略 `done: true` 那一次携带的 `value`**。`return 3` 的值不会被遍历出来，想让 `for..of` 拿到就得全部用 `yield`。
- 同样支持 iterable 的所有周边能力，如 spread：

```js
let sequence = [0, ...generateSequence()]; // 0, 1, 2, 3
```

### 5. 用 generator 实现 `Symbol.iterator`（最常用姿势）

- 手写 iterable 的 `range`（迭代器对象 + next 状态机）很啰嗦；把 `Symbol.iterator` 直接写成 generator 方法，代码大幅缩短：

```js
let range = {
  from: 1,
  to: 5,

  *[Symbol.iterator]() { // [Symbol.iterator]: function*() 的简写
    for (let value = this.from; value <= this.to; value++) {
      yield value;
    }
  }
};

alert([...range]); // 1,2,3,4,5
```

- 为什么可行：generator 自带 `.next()`，返回值形如 `{value, done}`，正是 `for..of` 所期望的。**这不是巧合——generator 就是为更容易实现 iterator 而加入语言的。**
- generator 还可以产出**无限序列**（如伪随机数流），此时消费端必须 `break`/`return`，否则死循环。

### 6. generator 组合：`yield*`

- `yield* gen` 把执行**委托**给另一个 generator：在 `gen` 上迭代，并把它 yield 的值**透明转发**给外部，就像这些值是外层 generator 自己 yield 的一样：

```js
function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

function* generatePasswordCodes() {
  yield* generateSequence(48, 57);  // 0..9
  yield* generateSequence(65, 90);  // A..Z
  yield* generateSequence(97, 122); // a..z
}

// 消费：0..9A..Za..z
```

- 效果与把内层代码内联展开完全一样，但**不需要额外内存存储中间结果**——流式拼接。

### 7. `yield` 是双向的：`next(arg)` 向 generator 内传值

- `generator.next(arg)` 把 `arg` 传入 generator，**成为当前 `yield` 表达式的求值结果**：

```js
function* gen() {
  let result = yield "2 + 2 = ?"; // (*) 先向外给出一个问题
  alert(result);                  // 外部回答被传回来，result = 4
}

let generator = gen();
let question = generator.next().value; // "2 + 2 = ?"（yield 给出的值）
generator.next(4);                     // 4 成为 (*) 行 yield 的结果
```

- **第一次 `next()` 必须无参**（传了也会被忽略），因为执行还没到达任何 `yield`。
- 外部不必立即回传，generator 会一直等——可以异步恢复：`setTimeout(() => generator.next(4), 1000)`。
- 多次往返像打乒乓球：每个 `next(value)`（除第一个）把 value 送入成为当前 `yield` 的结果，然后执行到下一个 `yield` 再把新结果抛出来。

### 8. `generator.throw(err)`：向 yield 处注入异常

- 外部不仅可以传值，还能传 error：调用 `generator.throw(err)`，`err` 会从对应的 `yield` 那一行**抛出来**。
- 如果 generator 内部有 `try..catch` 包住该 `yield`，就在内部捕获：

```js
function* gen() {
  try {
    let result = yield "2 + 2 = ?"; // (1) error 从这行抛出
  } catch(e) {
    alert(e); // 内部捕获
  }
}
let generator = gen();
generator.next();
generator.throw(new Error("The answer is not found in my database")); // (2)
```

- 内部没捕获 → 异常从 generator "掉出"到调用代码的 `generator.throw` 那一行（上例 `(2)`），可在外部 `try..catch`；外部也不捕获 → 沿调用链继续上抛，最终杀死脚本。

### 9. `generator.return(value)`：外部强制收尾

```js
const g = gen();
g.next();         // { value: 1, done: false }
g.return('foo');  // { value: "foo", done: true } —— 立即完成 generator
g.next();         // { value: undefined, done: true }
```

- 在已完成的 generator 上再调 `return()`，会再次返回该值。
- 不常用，适合"在特定条件下提前停止 generator"的场景。

### 10. 使用场景

- 现代 JS 中 generator 直接用得不多，但两个能力很独特：
  1. **执行过程中与调用代码双向交换数据**；
  2. **轻松创建可迭代对象**。
- 下一篇的 async generator 是它的重要延伸：`for await..of` 消费异步数据流（如网络分页拉取）。

## 与 Java 对比

| Java | JS | 说明 |
|---|---|---|
| 手写 `Iterator<T>`：类字段存状态 + `hasNext()`/`next()` | `function*` + `yield`，状态机由引擎自动生成 | 同样的迭代协议，JS 把样板代码消灭了 |
| 无对应（`Stream.generate` 只算半个） | 函数可**暂停/恢复**（协程能力） | Java 直到虚拟线程也没有语言级 yield 语义 |
| `Iterator.next()` 无参、单向取值 | `next(arg)` 双向通信，`yield` 是**表达式**能接收外部值 | Java 里要靠外部共享对象模拟，很别扭 |
| 迭代接口无此能力 | `generator.throw(err)` / `generator.return(v)` | 外部可往迭代中注异常、可提前终止 |
| `Stream` 的 `flatMap` 拼接流 | `yield*` 组合委托 | 都是零中间集合的流式拼接 |
| 增强 for 拿所有元素 | `for..of` **忽略 `done:true` 那次的 value** | 易踩坑：`return` 的值不进循环 |

## 一句话总结

`function*` 创建的 generator 调用时不执行、返回一个可推进的 generator 对象；`next()` 每次推进到下一个 `yield` 并返回 `{value, done}`；它天生可迭代（配 `for..of`/spread，也是实现 `Symbol.iterator` 的最佳姿势），`yield*` 做流式组合；`yield` 是双向通道——`next(arg)` 送值进来、`throw(err)` 注异常进来、`return(v)` 从外面收尾。
