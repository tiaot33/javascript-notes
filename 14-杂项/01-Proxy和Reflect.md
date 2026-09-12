# 01 Proxy 和 Reflect

> 原文：`zh.javascript.info/1-js/99-js-misc/01-proxy/article.md`
> **ES6 元编程核心**。包装任意对象并拦截其底层操作（读/写/删/in/调用/new……），Vue3 响应式就靠它。对应 Java 的动态代理/AOP。

## 知识点

### 1. 基本语法：透明包装器

```js
let proxy = new Proxy(target, handler);
```
- `target`：被包装的对象（可以是对象、函数、数组、类实例）。
- `handler`：捕捉器（trap）集合，如 `get` 拦截读取、`set` 拦截写入。
- **handler 为空时 proxy 是完全透明的**：`proxy.test = 5` 实际写入 target，读/迭代同样直接转发。
- Proxy 自身不存任何属性，是"奇异对象"。**代理后应处处用 proxy 替代 target，不要再引用原对象**。

### 2. 捕捉器对应规范"内部方法"，且受不变量约束
每个 trap 拦截一个规范内部方法：

| 内部方法 | 捕捉器 | 触发时机 |
|---|---|---|
| `[[Get]]` | `get` | 读属性 |
| `[[Set]]` | `set` | 写属性 |
| `[[HasProperty]]` | `has` | `in` 运算符 |
| `[[Delete]]` | `deleteProperty` | `delete` |
| `[[Call]]` | `apply` | 函数调用 |
| `[[Construct]]` | `construct` | `new` |
| `[[OwnPropertyKeys]]` | `ownKeys` | `Object.keys`、`for..in` 等 |

**不变量（invariant）**：如 `set` 成功必须返回 `true`（返回 falsy 触发 `TypeError`）；`[[GetPrototypeOf]]` 必须返回 target 的原型。不做怪事就不会违反。

### 3. `get`：实现默认值
```js
let numbers = new Proxy([0, 1, 2], {
  get(target, prop) {
    return (prop in target) ? target[prop] : 0; // 越界返回 0
  }
});
numbers[123]; // 0（常规数组是 undefined）
```
经典用途：词典查不到时返回原文、可观察对象（读时打日志）等。

### 4. `set`：写入校验
```js
numbers = new Proxy([], {
  set(target, prop, val) {
    if (typeof val == 'number') { target[prop] = val; return true; }
    return false; // TypeError
  }
});
numbers.push(1);      // ✓
numbers.push("test"); // ❌ TypeError
```
- 妙处：`push`/`unshift` 等内建方法内部走 `[[Set]]`，**不用重写它们就自动带上校验**。
- 别忘了成功时 `return true`。

### 5. `ownKeys` + `getOwnPropertyDescriptor`：控制枚举
过滤 `_` 开头的"私有"属性：
```js
user = new Proxy(user, {
  ownKeys(target) {
    return Object.keys(target).filter(key => !key.startsWith('_'));
  }
});
Object.keys(user); // name,age（_password 被过滤）
```
坑：`Object.keys` 只返回带 `enumerable` 标志的属性——`ownKeys` 返回**不存在的键**时，引擎会对每个键调 `[[GetOwnProperty]]` 查描述符，查不到就跳过。想让"虚拟键"被列出，需同时拦截：
```js
getOwnPropertyDescriptor(target, prop) {
  return { enumerable: true, configurable: true };
}
```

### 6. 受保护属性组合技 + 方法的 `this` 坑
完整保护 `_` 属性需要 4 个捕捉器：`get`/`set`/`deleteProperty` 抛错 + `ownKeys` 过滤。但 `get` 里有个关键细节：

```js
get(target, prop) {
  if (prop.startsWith('_')) throw new Error("Access denied");
  let value = target[prop];
  return (typeof value === 'function') ? value.bind(target) : value; // (*)
}
```
**为什么要 bind(target)**：方法 `user.checkPassword()` 被读出后，若 `this` 是 proxy，方法内部访问 `this._password` 会再次触发 `get` 抛错。绑定到原始 target 后方法内部操作绕过捕捉器。缺点：把未代理的原始对象泄漏给了方法，对象被多次代理时容易出意外。现代方案：类里直接用原生 `#私有字段`。

### 7. `has`：自定义 `in` 语义
```js
let range = new Proxy({ start: 1, end: 10 }, {
  has(target, prop) {
    return prop >= target.start && prop <= target.end;
  }
});
5 in range;  // true
50 in range; // false
```

### 8. `apply`：包装函数（比函数包装器更强）
回顾装饰器章的 `delay(f, ms)`——普通函数包装器会**丢失原函数的属性**（`sayHi.length` 从 1 变 0）。Proxy 版全转发：

```js
function delay(f, ms) {
  return new Proxy(f, {
    apply(target, thisArg, args) {
      setTimeout(() => target.apply(thisArg, args), ms);
    }
  });
}
sayHi = delay(sayHi, 3000);
sayHi.length; // 1 ✓（length 读取也被转发）
```
函数在 JS 里也是对象，`apply` 捕捉器拦截"被调用"这一操作，其余操作（读 `name`/`length`）自动转发。

### 9. Reflect：trap 的官方转发搭档
- `Reflect` 是内建对象，把 `[[Get]]`/`[[Set]]` 等规范内部方法暴露成可调用的函数：`Reflect.get(obj, prop)` ≡ `obj[prop]`，`Reflect.construct(F, args)` ≡ `new F(...args)`。
- **每个 Proxy 捕捉器都有同名同参数的 Reflect 方法**，转发操作的标准写法：
```js
user = new Proxy(user, {
  get(target, prop, receiver) {
    return Reflect.get(...arguments); // 等价 Reflect.get(target, prop, receiver)
  }
});
```

**为什么不用 `target[prop]` 直接转发？——getter + 继承场景会出 bug**：

```js
let user = { _name: "Guest", get name() { return this._name; } };
let userProxy = new Proxy(user, {
  get(target, prop, receiver) {
    return target[prop]; // (*)
  }
});
let admin = { __proto__: userProxy, _name: "Admin" };
admin.name; // "Guest"（期望 "Admin"！）
```
原因：`admin.name` 沿原型链找到 proxy，(*) 行 `target[prop]` 触发 getter 时 `this=target=user`，读到的是 user 的 `_name`。修复：用第三个参数 `receiver`（本次读取的真正发起者 admin）：

```js
get(target, prop, receiver) {
  return Reflect.get(target, prop, receiver); // getter 以 receiver 为 this 执行
}
```
getter 只能"被访问"不能 `call`，所以必须靠 `Reflect.get` 传递正确的 `this`。

### 10. Proxy 的三大局限
1. **内建对象的内部插槽（internal slot）**：`Map`/`Set`/`Date`/`Promise` 的数据存在 `[[MapData]]` 这类内部插槽，不走 `[[Get]]/[[Set]]`，Proxy 拦不到。代理后 `proxy.set(...)` 直接报错（方法内 `this=proxy` 没有插槽）。修复：`get` 里把函数属性 `value.bind(target)` 返回。**例外：`Array` 没有内部插槽**（历史原因），可放心代理。
2. **类的 `#私有字段`**：同样靠内部插槽实现，代理后 `getName()` 访问 `this.#name` 报错。修复同上（bind target），但会泄漏原始对象。
3. **`Proxy !== target`**：`===` 严格相等**无法被拦截**。用原对象做 `Set`/`Map` 的键后，proxy 查不到它：
```js
allUsers.has(user);               // true
allUsers.has(new Proxy(user, {})); // false
```
另外注意性能：即使最简单的代理，属性访问也比直接访问慢几倍——只对瓶颈对象才需要考虑。

### 11. 可撤销 Proxy：`Proxy.revocable`
```js
let { proxy, revoke } = Proxy.revocable(object, {});
proxy.data; // 正常
revoke();   // 切断 proxy 与 target 的所有内部引用
proxy.data; // Error
```
用途：把资源访问权发出去，随时可收回。配套模式：用 `WeakMap<proxy, revoke>` 存 revoke 函数（不阻碍 GC）。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `Proxy.newProxyInstance` + `InvocationHandler`（JDK 动态代理） | `new Proxy(target, handler)` | JS 版可代理**任何对象**（包括函数），不需要接口 |
| CGLIB/ByteBuddy 字节码增强 | 同左 | JS 语言内建，无需第三方库 |
| Spring AOP `@Cacheable`/`@Transactional` | `get`/`set`/`apply` 捕捉器 | 校验、缓存、日志、延迟等切面逻辑手写即可 |
| `Method.invoke(obj, args)`、`Field.get(obj)`（反射 API） | `Reflect.get/set/construct` | 把运算符（`new`/`delete`/`.`）变成函数调用 |
| AOP 里 `proceed()` 转发给目标 | `return Reflect.get(...arguments)` | 官方推荐的透明转发写法 |
| —— | `Proxy.revocable` 可撤销代理 | Java 无内建对应，近似于手动使引用失效 |

## 一句话总结
`new Proxy(target, handler)` 包装对象并用捕捉器拦截 `[[Get]]/[[Set]]/[[Call]]` 等底层操作，实现默认值、写入校验、键过滤、受保护属性、`in` 语义、函数装饰等元编程能力；handler 为空时完全透明；转发务必用同名同参的 `Reflect.*`（getter 继承场景靠 `receiver` 才能拿到正确的 `this`）；记住三个坑：内建对象内部插槽和类私有字段代理后会断（需 `bind(target)` 救场）、`===` 拦不住（proxy 与 target 是两个对象）、`set` 成功必须 `return true`。
