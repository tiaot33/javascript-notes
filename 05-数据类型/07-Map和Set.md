# 07 Map 和 Set（映射和集合）

> 原文：`zh.javascript.info/1-js/05-data-types/07-map-set/article.md`
> ES6 新增数据结构。

## 知识点

### 1. Map：允许**任意类型**键的键值集合
API：
- `new Map()` / `new Map(iterable)`（可用 `[key, value]` 对数组初始化）
- `map.set(key, value)` —— 返回 map 本身，**可链式调用**
- `map.get(key)` —— 不存在返回 `undefined`
- `map.has(key)` / `map.delete(key)` / `map.clear()` / `map.size`（属性！）

```js
let map = new Map();
map.set('1', 'str1').set(1, 'num1').set(true, 'bool1');

map.get(1);   // 'num1' —— Map 保留键的类型
map.get('1'); // 'str1' —— 与数字 1 是不同的键！
map.size;     // 3
```

**与 Object 的关键区别**：普通对象会把键转成字符串，Map 保留原始类型。

⚠️ `map[key] = 2` 语法上可行，但那是把 map 当普通对象用，丢掉了 Map 的优势——**必须用 set/get/has 方法**。

### 2. 对象作键（Map 最重要的能力）
```js
let john = { name: "John" };
let visitsCountMap = new Map();
visitsCountMap.set(john, 123);
visitsCountMap.get(john); // 123
```

普通对象做不到：
```js
let visitsCountObj = {};
visitsCountObj[john] = 123;   // 键被转成 "[object Object]"！
visitsCountObj[ben] = 234;    // 把上面覆盖了
```

**键比较算法**：SameValueZero —— 类似 `===`，但认为 `NaN` 等于 `NaN`（所以 NaN 也能作键）。不可自定义。

### 3. Map 的迭代：保持插入顺序
- `map.keys()` / `map.values()` / `map.entries()` —— 都返回可迭代对象。
- `for..of map` 默认等价于 `map.entries()`。
- **迭代顺序 = 插入顺序**（普通 Object 做不到这一点，整数键还会被排序）。

```js
let recipeMap = new Map([
  ['cucumber', 500],
  ['tomatoes', 350],
  ['onion', 50]
]);

for (let entry of recipeMap) { /* [key, value] */ }

recipeMap.forEach((value, key, map) => { // 注意参数顺序：value 在前！
  alert(`${key}: ${value}`);
});
```
⚠️ `map.forEach` 回调参数是 `(value, key, map)`，value 在前。

### 4. Object ⇄ Map 互转
```js
// Object → Map
let obj = { name: "John", age: 30 };
let map = new Map(Object.entries(obj)); // entries 返回 [["name","John"],["age",30]]

// Map → Object
let obj2 = Object.fromEntries(map); // 或 Object.fromEntries(map.entries())，效果相同
```

### 5. Set：唯一值的集合
- `new Set(iterable?)`
- `set.add(value)` —— 返回 set 本身；**重复添加无效**（这就是去重原理）
- `set.delete(value)` → bool；`set.has(value)` → bool；`set.clear()`；`set.size`

```js
let set = new Set();
set.add(john); set.add(pete); set.add(mary); set.add(john); set.add(mary);
set.size; // 3 —— 重复的不算
```
- 内部对唯一性检查有优化；比"数组 + find 查重"快得多。

### 6. Set 的迭代
- 保持插入顺序；`for..of` 和 `forEach` 都可以。

```js
set.forEach((value, valueAgain, set) => { ... });
```
- 奇怪的兼容设计：回调里 value 出现**两次**（为了和 Map 的 `(value, key)` 签名兼容，方便 Map/Set 互换）。
- `set.keys()` / `set.values()`（两者相同）、`set.entries()`（返回 `[value, value]`）—— 都为兼容 Map 而存在。

### 7. 顺序总结
Map 和 Set 都**按插入顺序迭代**，不能重排，也不能按编号取值。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `HashMap<K,V>` 任意对象键 | `Map` 任意类型键 | 终于对等了；JS 普通 Object 键只能是 string/symbol |
| `map.put(k,v)` / `get(k)` / `containsKey(k)` / `remove(k)` | `set/get/has/delete` | JS 的 get 不存在时返回 undefined |
| `map.size()` | `map.size`（属性） | 又是属性 |
| `LinkedHashMap` 保插入序 | Map 天生保插入序 | JS Map 迭代永远按插入序 |
| `keySet()/values()/entrySet()` | `keys()/values()/entries()` | JS 返回可迭代对象，可直接 for..of |
| 键相等用 equals/hashCode，可重写 | SameValueZero，不可自定义 | JS 对象键按**引用**比较，无 equals 概念 |
| `HashSet` / `LinkedHashSet` | `Set`（保插入序） | add 重复无效 |
| `Collections.unmodifiable...` 无对应 | — | — |

**什么时候用 Map 而不是 Object**（原文思想的延伸）：
- 键不是字符串（数字、对象、NaN……）→ 必须 Map。
- 需要保持插入顺序、频繁增删、需要 `size` → Map 更合适。
- 存储结构化记录（固定字段）→ 普通 Object 更方便（字面量、点访问、JSON 直接序列化）。

## 一句话总结
Map 是键可为任意类型（包括对象）的键值集合，用 `set/get/has/delete` 操作（别用 `map[key]`），保持插入顺序，迭代用 `keys/values/entries` 或 `for..of`；Set 是唯一值集合，`add` 重复无效；两者都是 ES6 新增、按插入序迭代；`Object.entries` / `Object.fromEntries` 负责 Object 与 Map 互转。
