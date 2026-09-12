# 09 Object.keys，values，entries

> 原文：`zh.javascript.info/1-js/05-data-types/09-keys-values-entries/article.md`

## 知识点

### 1. 通用约定
`keys()` / `values()` / `entries()` 是各数据结构的通用迭代约定，`Map`、`Set`、`Array` 都支持。普通对象（plain object）也有对应物，但语法不同。

### 2. 普通对象的三个静态方法
- `Object.keys(obj)` —— 所有键组成的数组
- `Object.values(obj)` —— 所有值组成的数组
- `Object.entries(obj)` —— 所有 `[key, value]` 对组成的数组

```js
let user = { name: "John", age: 30 };

Object.keys(user);    // ["name", "age"]
Object.values(user);  // ["John", 30]
Object.entries(user); // [["name","John"], ["age",30]]

for (let value of Object.values(user)) { ... }
```

### 3. 与 Map 方法的两个区别
| | Map | Object |
|---|---|---|
| 调用语法 | `map.keys()` | `Object.keys(obj)`（静态方法，不是 `obj.keys()`） |
| 返回值 | 可迭代对象 | **真正的数组** |

- 为什么是静态方法？**灵活性**：对象是所有复杂结构的基础，你自己创建的对象（如 `data`）可能定义了自己的 `data.values()`；`Object.values(data)` 依然可用，互不冲突。
- 返回真数组是历史原因（这些方法出现时还没有迭代器协议）——好处是可以直接接 `map/filter` 等数组方法。

### 4. 忽略 Symbol 键
和 `for..in` 一样，这三个方法**忽略 Symbol 类型的键**。

- 只要 Symbol 键：`Object.getOwnPropertySymbols(obj)`
- 要全部键（字符串 + Symbol）：`Reflect.ownKeys(obj)`

### 5. 对象转换链（核心技巧）
对象没有 `map`/`filter` 等数组方法，但可以"对象 → 数组 → 转换 → 对象"：

```js
let prices = { banana: 1, orange: 2, meat: 4 };

let doublePrices = Object.fromEntries(
  Object.entries(prices).map(entry => [entry[0], entry[1] * 2])
);

doublePrices.meat; // 8
```

三步：
1. `Object.entries(obj)` 把对象拆成 `[key, value]` 数组。
2. 用数组方法（`map`/`filter`...）转换。
3. `Object.fromEntries(array)` 转回对象。

可以借此建立强大的转换链，用一两次就上手。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `map.keySet()` / `values()` / `entrySet()` | `Object.keys/values/entries(obj)` | JS 普通对象的版本是静态方法，返回真数组 |
| `stream().map(...).collect(toMap(...))` | `Object.fromEntries(entries.map(...))` | 对象转换链 ≈ Java 里 map→stream→map 的写法，但更轻 |
| Bean 属性遍历（反射） | `Object.keys(obj)` | JS 直接内建，无需反射 |

## 一句话总结
普通对象用静态方法 `Object.keys/values/entries(obj)` 获取键/值/键值对的**真数组**（区别于 Map 的实例方法返回可迭代对象）；它们忽略 Symbol 键；对象想用数组方法时走 `Object.entries` → 数组转换 → `Object.fromEntries` 的转换链。
