# 04 原型方法，没有 __proto__ 的对象

> 原文：`zh.javascript.info/1-js/08-prototypes/04-prototype-methods/article.md`
> 收尾篇：`__proto__` 已过时，给出**现代的原型读写 API**；顺带讲清为什么 `__proto__` 会成为 bug 甚至安全漏洞（原型污染），以及用 `Object.create(null)` 造"纯字典"对象的技巧。做后端的尤其要看第 5、6 节。

## 知识点

### 1. 现代的原型 API（取代 `__proto__`）
| 目的 | 现代写法 | 说明 |
|---|---|---|
| 读原型 | `Object.getPrototypeOf(obj)` | 等价于 `obj.__proto__` 的 getter |
| 改原型 | `Object.setPrototypeOf(obj, proto)` | 等价于 `obj.__proto__ = proto` 的 setter |
| 以指定原型创建对象 | `Object.create(proto, [descriptors])` | 创建一个 `[[Prototype]]` 为 `proto` 的空对象，可选第二参数用**属性描述符**添加属性 |
| 字面量指定原型 | `{ __proto__: proto, a: 1 }` | `__proto__` **唯一**不被反对的用法，且能同时写多个属性 |

```js
let animal = { eats: true };

let rabbit = Object.create(animal);          // 同 { __proto__: animal }
Object.getPrototypeOf(rabbit) === animal;    // true
Object.setPrototypeOf(rabbit, {});           // 原型改为 {}

// 带属性描述符（格式同 07-对象属性配置）
let rabbit2 = Object.create(animal, {
  jumps: { value: true }   // 注意：描述符方式新建属性，未写的标志默认全 false
});
rabbit2.jumps; // true
```

- `obj.__proto__` 作为 getter/setter 的用法已被移到规范附录 B（仅浏览器必须支持），**新代码不要再用**。

### 2. 用 `Object.create` 做"完整"浅拷贝
```js
let clone = Object.create(
  Object.getPrototypeOf(obj),            // 原型也一样
  Object.getOwnPropertyDescriptors(obj)  // 所有自身属性的描述符
);
```
- 相比 `for..in` 逐个复制或 `Object.assign`：连**不可枚举属性、symbol 键、getter/setter、各属性标志、`[[Prototype]]`** 都一并复制，是真正精确的克隆。
- 仍是**浅拷贝**：嵌套对象只复制引用。

### 3. 原型简史（为什么有这么多种方式）
| 阶段（按教程） | 事件 |
|---|---|
| 起初 | 只有构造函数的 `F.prototype`，是设置原型的唯一手段 |
| 2012 | `Object.create` 进入标准：能"以某原型创建对象"，但不能读写已有对象的原型；浏览器私自实现了非标准的 `__proto__` 访问器补这个缺口 |
| 2015（ES6） | `Object.getPrototypeOf / setPrototypeOf` 进入标准；`__proto__` 因已被广泛实现而收进附录 B（非浏览器环境可选支持） |
| 2022 | 对象字面量里的 `__proto__: ...` 被正式认可（移出附录 B）；`obj.__proto__` 读写仍留在附录 B |

> 注：严格说 `Object.create` 和 `Object.getPrototypeOf` 在 2009 年的 ES5 就已进入标准，ES6 新增的是 `setPrototypeOf`。教程的年份只是概述，记住演进顺序和结论即可。

### 4. 性能警告：不要"即时"修改已有对象的原型
- 技术上任何时候都能 `setPrototypeOf`，但**正确用法是创建时设一次，之后不再动**：`rabbit` 继承自 `animal`，就一直如此。
- 引擎对属性访问做了大量依赖原型链稳定的内部优化，运行中改原型会**破坏这些优化，非常慢**，影响的还不止这一次操作。
- 需要不同原型 → 用 `Object.create(proto)` / `{ __proto__: proto }` 新建对象，而不是改旧对象。

### 5. `__proto__` 作为字典键的 bug（原型污染的根源）
用普通对象当 Map 存**用户提供的键**时：

```js
let obj = {};
let key = prompt("What's the key?", "__proto__");
obj[key] = "some value";
alert(obj[key]); // [object Object]，不是 "some value"！
```
- 原因：`__proto__` 不是 `obj` 自己的属性，而是 **`Object.prototype` 上的访问器属性**。`obj["__proto__"] = "..."` 触发的是继承来的 setter，它只接受对象或 `null`，字符串被忽略；读的时候 getter 返回的是当前原型 `Object.prototype`。
- 更严重的情形：如果存的值是对象，**原型真的会被换掉**，后续所有属性查找行为都变，程序以完全意料之外的方式出错。
- 这类 bug 极难发现，在**服务端 JS**（Node）处理用户输入（如递归合并 JSON、解析 query 参数）时会演变成安全漏洞——即所谓 **prototype pollution（原型污染）**。
- 同理，`obj.toString = 用户输入` 之类对内建方法名的赋值也会出意外。

### 6. 三种解法
1. **用 `Map`**：`map.set(key, value)`，键是什么都无所谓，彻底规避。（见 05-数据类型/07-Map和Set。）但对象字面量语法更简洁，很多场景还是想用对象。
2. **"very plain" 对象 / 纯字典**：`Object.create(null)` 或 `{ __proto__: null }`，`[[Prototype]]` 是 `null`，不继承 `Object.prototype`，也就没有 `__proto__` 访问器，`"__proto__"` 只是个普通字符串键。

```js
let obj = Object.create(null);   // 或 { __proto__: null }
obj["__proto__"] = "some value";
obj["__proto__"];                // "some value" ✓
```
   - 代价：没有任何内建方法，`alert(obj)`、`obj.toString()`、`obj.hasOwnProperty()` 都会报错。
   - 但 `Object.keys(obj)`、`Object.values`、`Object.entries` 等**静态方法照常可用**，因为它们挂在 `Object` 上而不是 `Object.prototype` 上。判断自身属性可用 `Object.hasOwn(obj, key)`（ES2022）。
   - 需要 `toString` 时可用描述符补一个不可枚举的（练习题）：

```js
let dictionary = Object.create(null, {
  toString: {                                  // 描述符定义，默认 enumerable: false
    value() { return Object.keys(this).join(); }
  }
});
dictionary.apple = "Apple";
dictionary.__proto__ = "test";                 // 普通键
for (let key in dictionary) alert(key);        // apple, __proto__（toString 不出现）
alert(dictionary);                             // "apple,__proto__"
```
3. 教程之外的实战做法：对用户键做白名单/前缀处理，合并 JSON 时显式跳过 `__proto__`、`constructor`、`prototype` 键。

### 7. 练习题要点：调用方式决定 `this`
```js
function Rabbit(name) { this.name = name; }
Rabbit.prototype.sayHi = function() { alert(this.name); };
let rabbit = new Rabbit("Rabbit");

rabbit.sayHi();                        // Rabbit
Rabbit.prototype.sayHi();              // undefined
Object.getPrototypeOf(rabbit).sayHi(); // undefined
rabbit.__proto__.sayHi();              // undefined
```
- 四个调用找到的是**同一个函数**，但 `this` 是点号前的对象：后三个的 `this` 都是 `Rabbit.prototype`，它没有 `name`。再次印证第 1 篇的规则。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `obj.getClass().getSuperclass()`，只读 | `Object.getPrototypeOf(obj)` 读，`Object.setPrototypeOf` 可写 | JS 连"父类"都能运行时改，但性能代价大，应视为禁手 |
| 无对应（不能"以某对象为父"直接造对象） | `Object.create(proto)` / `{ __proto__: proto }` | 对象直接派生对象，不经过类 |
| `clone()` / 拷贝构造器，字段级复制 | `Object.create(getPrototypeOf(obj), getOwnPropertyDescriptors(obj))` | JS 一行拿到含属性标志、访问器和原型的精确浅拷贝 |
| `HashMap<String, V>` 任意字符串键都安全 | 普通对象当字典时 `"__proto__"`、`"toString"` 等键会撞上原型 | 用 `Map` 或 `Object.create(null)` 才等价于 `HashMap` |
| Jackson 多态反序列化的 gadget 漏洞 | prototype pollution：合并用户 JSON 时污染 `Object.prototype` | 都是"用户输入改变了对象模型"的后端安全问题，Node 生态里 lodash `merge` 等的 CVE 即此类 |
| `Object` 的方法都是实例方法 | 常用工具是 `Object.xxx(obj)` 静态方法（`keys/values/entries/assign`） | 静态方法不依赖原型，所以在 `Object.create(null)` 对象上依然可用 |

## 一句话总结
读写原型用 `Object.getPrototypeOf / setPrototypeOf`，以指定原型建对象用 `Object.create(proto, [descriptors])` 或字面量 `{ __proto__: proto }`（`__proto__` 唯一被认可的用法），`obj.__proto__` 读写已过时；原型应在创建时设定一次，运行中改原型会毁掉引擎优化；`Object.create(getPrototypeOf(obj), getOwnPropertyDescriptors(obj))` 能做含标志、访问器和原型的精确浅拷贝；普通对象当字典存用户键时 `"__proto__"` 会触发继承自 `Object.prototype` 的 setter，轻则丢数据重则原型污染，改用 `Map`，或用 `Object.create(null)` / `{ __proto__: null }` 创建没有原型的"纯字典"对象（没有 `toString` 等内建方法，但 `Object.keys` 等静态方法照常可用）。
