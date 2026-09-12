# 09 装饰器模式和转发，call/apply

> 原文：`zh.javascript.info/1-js/06-advanced-functions/09-call-apply-decorators/article.md`
> JS 高阶函数的经典玩法：**包装器（wrapper）+ call/apply 转发调用**。装饰器/防抖/节流都靠这套。

## 知识点

### 1. 装饰器：改变函数行为的包装器
不修改原函数，用包装器给它加功能（这里加缓存）：

```js
function cachingDecorator(func) {
  let cache = new Map();
  return function(x) {
    if (cache.has(x)) return cache.get(x);
    let result = func(x);
    cache.set(x, result);
    return result;
  };
}
slow = cachingDecorator(slow);
```
- 好处：可复用、主函数不受污染、多个装饰器可组合（每条对应 SOLID/开闭原则的思路）。

### 2. 方法装饰需要 `func.call` 传递 `this`
上面的装饰器包装**对象方法**时出错：

```js
worker.slow = cachingDecorator(worker.slow);
worker.slow(2); // ❌ Cannot read property 'someMethod' of undefined
```
原因：包装器里 `func(x)` 直接调用，`this` 是 `undefined`，原方法依赖的 `this.someMethod()` 丢失。

**修复**：用内建方法在调用时显式设定 `this`：

```js
let result = func.call(this, x); // this=worker，参数 x，正确传递
```

`func.call(context, arg1, arg2, ...)`：以 `context` 为 `this` 运行 func。

```js
function sayHi() { alert(this.name); }
sayHi.call(user);  // this=user
sayHi.call(admin); // this=admin
```

### 3. 多参数转发：`func.call(this, ...arguments)`
包装器里拿到所有参数，配合 hash 函数生成缓存 key：

```js
function cachingDecorator(func, hash) {
  let cache = new Map();
  return function() {
    let key = hash(arguments);                    // 参数组合作 key
    if (cache.has(key)) return cache.get(key);
    let result = func.call(this, ...arguments);   // 全部参数 + this 转发
    cache.set(key, result);
    return result;
  };
}
function hash(args) { return args[0] + ',' + args[1]; }
```

### 4. `func.apply`
`func.apply(context, argsArray)`：与 `call` 作用相同，但参数用**类数组**整体传：

```js
func.call(context, ...args);
func.apply(context, args); // 两个等价
```
- spread `...` 接受**可迭代对象**，`apply` 接受**类数组**；真数组两者皆可，多数引擎对 `apply` 有优化可能更快。
- **调用转发（call forwarding）**的最简形式：

```js
let wrapper = function() {
  return func.apply(this, arguments);
};
```
外部调用包装器与调用原函数无法区分——这是装饰器透明性的关键。

### 5. 方法借用（method borrowing）
`arguments` 是类数组但没有数组方法，借用数组的方法：

```js
function hash() {
  alert( [].join.call(arguments) ); // 1,2 —— 从数组借 join 用到 arguments 上
}
```
- 原理：`arr.join` 内部只操作 `this[0..length]`，对任何类数组 `this` 都成立（很多原生方法故意这么设计）。
- 常见做法：取数组方法应用于 `arguments`；更现代的做法是用 rest 参数拿真数组。

### 6. 装饰器会丢失函数属性
- 包装替换后，原函数挂的属性（如 `func.calledCount`）就访问不到了。
- 如需保留属性访问，要用 `Proxy`（后面章节）包装。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 装饰器模式（`new BufferedInputStream(in)`）、AOP 动态代理 | 高阶函数包装器 | JS 用纯函数实现，不用接口和代理类 |
| `Method.invoke(obj, args)`（反射） | `func.call(this, ...args)` / `func.apply(this, args)` | 显式指定 receiver 调用方法 |
| `InvocationHandler.invoke` 里转发 | call forwarding | apply 一行实现完美转发 |
| Spring AOP `@Cacheable` | cachingDecorator | 手写版缓存切面 |
| —— | 方法借用 `[].join.call(arguments)` | Java 没有这种"把方法挪到别的对象上执行"的玩法 |

## 一句话总结
装饰器 = 不改变原函数代码的包装器，可以叠加缓存/防抖等"切面"；对象方法转发必须用 `func.call(this, ...args)` 或 `func.apply(this, argsArray)` 显式带上 `this`，否则 `this` 变成 `undefined` 直接报错；`call` 与 `apply` 只差参数形式（列表 vs 类数组）；`[].join.call(arguments)` 这类**方法借用**利用原生方法只依赖 `this` 类数组形态的特性，把数组方法用到 `arguments` 上。
