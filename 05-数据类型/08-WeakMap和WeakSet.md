# 08 WeakMap 和 WeakSet（弱映射和弱集合）

> 原文：`zh.javascript.info/1-js/05-data-types/08-weakmap-weakset/article.md`
> ES6 新增。前置知识：垃圾回收章节（对象"可达"就留在内存中）。

## 知识点

### 1. 问题背景：集合会阻止垃圾回收
对象放进数组或普通 `Map` 后，只要集合活着，对象就"可达"，**不会被回收**：

```js
let john = { name: "John" };
let map = new Map();
map.set(john, "...");
john = null; // 覆盖引用
// john 对象仍然活着！因为它是 map 的键，还能通过 map.keys() 拿到
```

`WeakMap` 的根本区别：**不阻止垃圾回收对键对象的回收**。

### 2. WeakMap 的规则
- **键必须是对象**（原始值不行）：

```js
let weakMap = new WeakMap();
weakMap.set({}, "ok");        // 正常
weakMap.set("test", "Whoops"); // Error：键必须是对象
```

- 当键对象**失去所有其他引用**时，它和它的条目会被自动从内存（和 WeakMap）中清除：

```js
let john = { name: "John" };
weakMap.set(john, "...");
john = null; // john 被从内存中删除了，WeakMap 里的条目也随之消失
```

### 3. WeakMap 的 API 限制
只有 4 个方法：
- `weakMap.get(key)` / `set(key, value)` / `delete(key)` / `has(key)`

**不支持**：迭代、`keys()`、`values()`、`entries()`、`size`、`clear()`。

**为什么？** 垃圾回收的时机由引擎决定，无法预知——可能立即清理，也可能等批量删除时才清理。所以 WeakMap 的"当前内容"在技术上是不确定的，干脆不提供遍历能力。

### 4. 使用案例一：额外数据存储（主要场景）
给"属于别人代码"的对象（第三方库、框架管理的对象）附加数据，且希望数据**与对象共存亡**：

```js
// 📁 visitsCount.js
let visitsCountMap = new WeakMap(); // user => 访问次数

function countUser(user) {
  let count = visitsCountMap.get(user) || 0;
  visitsCountMap.set(user, count + 1);
}

// 📁 main.js
let john = { name: "John" };
countUser(john);
john = null; // 用户离开：WeakMap 里的计数自动清除，无需手动清理
```
用普通 `Map` 的话，用户对象会永远留在 map 里（内存泄漏），复杂架构中手动清理是繁重任务。

### 5. 使用案例二：缓存
```js
// 📁 cache.js
let cache = new WeakMap();

function process(obj) {
  if (!cache.has(obj)) {
    let result = /* 对 obj 的昂贵计算 */ obj;
    cache.set(obj, result);
  }
  return cache.get(obj);
}
```
- 同一对象重复调用直接取缓存。
- 对象不再需要时 `obj = null`，缓存条目**自动清除**——用 `Map` 则缓存会一直占着内存。

### 6. WeakSet
- 只能存**对象**（不能是原始值）。
- 对象在其他地方可达时才留在 WeakSet 中。
- 只支持 `add` / `has` / `delete`；**没有 size、keys()，不可迭代**。
- 适用"是/否"标记类场景：

```js
let visitedSet = new WeakSet();
visitedSet.add(john);           // 记录"访问过"
visitedSet.has(john);           // true
visitedSet.has(mary);           // false
john = null;                    // 自动从集合中清除
```

### 7. 总结定位
- WeakMap/WeakSet 是"主要存储之外的**辅助**数据结构"。
- 优点：弱引用，不阻止回收；代价：不可迭代、无 size/keys/clear。
- 一旦对象从主存储删除且仅被 WeakMap/WeakSet 引用，它就自动消失。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `WeakHashMap`（键是弱引用） | `WeakMap` | 思想一致：键不再被引用时条目被 GC 移除 |
| `WeakHashMap` 有 `size()` / 可遍历（但不可靠） | `WeakMap` **完全不可迭代**、无 size | JS 更彻底：既然 GC 时机不确定，干脆不提供遍历 |
| 键可以是任何对象 | 键**必须**是对象（不能原始值） | 一致（原始值本来就没有"引用"可言） |
| Guava Cache / 手动缓存清理 | WeakMap 做对象级缓存 | 对象回收即缓存失效 |
| 无直接对应 | WeakSet | 类似 `Collections.newSetFromMap(new WeakHashMap<>())` |

## 一句话总结
WeakMap（键必须是对象）和 WeakSet（只能存对象）对条目持弱引用：对象一旦在外部不可达，连同它的条目一起被垃圾回收自动清除；代价是只有 get/set/has/delete，不能迭代也没有 size；典型用途是给外部对象挂"共存亡"的附加数据（如访问计数）和自动失效的缓存。
