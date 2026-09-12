# 05 BigInt

> 原文：`zh.javascript.info/1-js/99-js-misc/05-bigint/article.md`
> **ES2020 新增的第 8 种数据类型**：任意精度整数。解决 `number`（IEEE 754 双精度）超过 `2^53 - 1` 后失真的问题——典型场景：后端 64 位雪花 ID、加密计算。

## 知识点

### 1. 创建方式（两种）
```js
const bigint = 1234567890123456789012345678901234567890n; // 字面量加 n 后缀
const sameBigint = BigInt("1234567890123456789012345678901234567890"); // 从字符串
const bigintFromNumber = BigInt(10); // 与 10n 相同
```
为什么需要它：常规 `number` 安全整数上限是 `Number.MAX_SAFE_INTEGER`（`2^53 - 1`），超过后精度丢失：
```js
9007199254740991 + 1 === 9007199254740991 + 2 // true（精度已丢）
```

### 2. 数学运算
```js
1n + 2n;  // 3n
5n / 2n;  // 2n —— 向零取整，没有小数部分
```
- 所有 bigint 运算结果仍是 bigint。

### 3. 不能与 number 混算（重要）
```js
1n + 2; // ❌ TypeError: Cannot mix BigInt and other types
```
必须显式转换：
```js
let bigint = 1n, number = 2;
bigint + BigInt(number); // 3n
Number(bigint) + number; // 3
```
- 转换**静默进行、绝不报错**，但 bigint 超出 number 容量时会被**截断**——需谨慎。
- **一元加号 `+` 不支持 bigint**（`+1n` 报错），转数字请用 `Number(bigint)`。

### 4. 比较运算：宽松混合，严格区分
```js
2n > 1n;  // true
2n > 1;   // true —— <、> 可与 number 混比

1 == 1n;  // true  —— == 会做类型转换
1 === 1n; // false —— === 区分类型，number 和 bigint 是不同类型
```

### 5. 布尔语义与 number 一致
```js
if (0n) { /* 不执行 */ } // 0n 为 falsy，其余 bigint 为 truthy
1n || 2; // 1n
0n || 2; // 2
```

### 6. Polyfill 难题与 JSBI 方案
- bigint 无法优雅 polyfill：`+`、`-`、`/` 等运算符行为不同，polyfill 得分析并替换所有运算符，性能代价大。
- 实际方案：写代码时用 **JSBI 库**（方法式 API：`JSBI.add(a, b)`），支持 bigint 的环境再通过 Babel 插件把 JSBI 调用编译成原生 bigint。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `BigInteger`（对象，用 `.add()`/`.subtract()` 方法运算） | `BigInt`（**原始类型**，直接用 `+`/`-`/`*` 运算符） | JS 的写法比 Java 自然得多 |
| `long`（64 位精确整数） | 无直接对应——`number` 只有 53 位安全精度 | JS 一直缺"long"，BigInt 补上了这个洞（且不限 64 位） |
| `Long` vs `BigInteger` 两种类型 | `number` vs `bigint`，同样不能混算 | 两边都要求显式转换 |
| `new BigInteger("...")` | `BigInt("...")` / `...n` 字面量 | JS 有字面量语法，更方便 |
| —— | `1 == 1n` 为 true 但 `1 === 1n` 为 false | JS 特有的宽松/严格相等差异，Java 无对应 |

## 一句话总结
`BigInt` 是任意精度整数类型（字面量加 `n` 或 `BigInt()` 创建），用于突破 `number` 的 `2^53 - 1` 安全上限（如后端 64 位 ID）；运算结果恒为 bigint、除法向零取整；**禁止与 number 混合运算**（需 `BigInt()`/`Number()` 显式互转，且转 number 可能静默截断）；比较上 `<`/`>` 可混比、`==` 相等但 `===` 不等；布尔语义同 number（`0n` 为假）；无好用 polyfill，老环境用 JSBI 库写方法式代码再编译成原生。
