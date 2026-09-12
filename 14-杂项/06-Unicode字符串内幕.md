# 06 Unicode —— 字符串内幕

> 原文：`zh.javascript.info/1-js/99-js-misc/06-unicode/article.md`
> 处理 emoji、生僻字前的必修课：JS 字符串是 **UTF-16** 编码，"一个字符"不一定占一个长度。Java 程序员应该倍感亲切——Java 的 `String` 同样是 UTF-16。

## 知识点

### 1. 三种 Unicode 转义写法
| 写法 | 范围 | 例子 |
|---|---|---|
| `\xXX` | 2 位十六进制（仅前 256 个字符） | `"\x7A"` → `z` |
| `\uXXXX` | 4 位十六进制（`0000`–`FFFF`） | `"\u00A9"` → `©` |
| `\u{X…XXXXXX}` | **ES6 新增**，1–6 位十六进制，可到 `10FFFF` | `"\u{1F60D}"` → 😍 |

只有第三种能直接书写超出 `FFFF` 的字符。

### 2. 代理对（surrogate pair）：为什么 emoji 的 length 是 2
- JS 字符串基于 UTF-16，每个"码元（code unit）"2 字节，只有 65536 种组合，放不下全部 Unicode。
- 超出 `U+FFFF` 的稀有字符用**一对 2 字节码元**表示，即代理对。副作用：

```js
'𝒳'.length; // 2（数学符号 X）
'😂'.length; // 2
'𩷶'.length; // 2（生僻中文字）

'𝒳'[0]; // 乱码——代理对的前半截，单独无意义
```
- 判定范围：高代理 `0xD800..0xDBFF`，低代理 `0xDC00..0xDFFF`（规范专为代理对预留）。
- JS 诞生时代理对概念还不存在，语言层面的 `length`、下标按**码元**计而非**字符**计。

### 3. 正确处理代理对的 API
| 老 API（码元视角） | 新 API（码点视角，ES6） |
|---|---|
| `str.charCodeAt(i)` | `str.codePointAt(i)` |
| `String.fromCharCode(n)` | `String.fromCodePoint(n)` |

```js
'𝒳'.charCodeAt(0).toString(16);   // d835（只读到前半截）
'𝒳'.codePointAt(0).toString(16);  // 1d4b3（完整字符 ✓）
```
> 补充（可迭代一章）：`for..of` 和 `[...str]` 展开也按码点迭代，能正确遍历 emoji——比按下标循环安全。

### 4. 警告：不能随意位置切字符串
```js
'hi 😂'.slice(0, 4); // "hi �" —— 切断了代理对，留下半个乱码
```
按码元下标 `slice`/`substring` 可能把代理对拦腰截断。需要"按字符数"截取时先转码点数组：`[...str].slice(0, n).join('')`。

### 5. 变音符号与规范化（normalization）
- 很多字符 = **基础字符 + 组合记号**，如 `'S' + '\u0307'`（上方的点）显示为 `Ṡ`，可无限叠加：`'S\u0307\u0323'`。
- 由此产生陷阱：**视觉相同的字符，底层 Unicode 组合可能不同**，直接比较失败：
```js
let s1 = 'S\u0307\u0323'; // S + 上点 + 下点
let s2 = 'S\u0323\u0307'; // S + 下点 + 上点
s1 == s2; // false（肉眼看一模一样）
```
- 解法：`str.normalize()` 转成统一规范形式再比较：
```js
s1.normalize() === s2.normalize(); // true
'S\u0307\u0323'.normalize().length; // 1 —— 足够常见的组合被合并成单码点 \u1e68
```
- 用户输入检索、去重、做哈希键之前，记得先 normalize。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| `String` 同样是 **UTF-16** | 同样 UTF-16 | 两套语言的坑一模一样！ |
| `"😂".length() == 2` | `'😂'.length === 2` | 都按码元计数 |
| `charAt(i)` vs `codePointAt(i)` | `charCodeAt(i)` vs `codePointAt(i)` | 连 API 命名都几乎相同 |
| `new String(Character.toChars(cp))` | `String.fromCodePoint(cp)` | 码点转字符串 |
| `java.text.Normalizer.normalize(s, NFD/NFC)` | `str.normalize()`（默认 NFC） | 规范化思路一致 |
| `"\u0041"` 仅支持 4 位 | `\uXXXX` + ES6 的 `\u{1F60D}` | JS 的花括号写法更灵活 |

## 一句话总结
JS 字符串是 UTF-16 编码：`\xXX`/`\uXXXX`/`\u{...}`（ES6）三种转义中仅花括号形式能写全部码位；超出 `U+FFFF` 的字符（emoji、生僻字）以**代理对**存储，`length` 和下标按码元计所以 `'😂'.length === 2`，半截代理对是乱码、随意 `slice` 会切出垃圾——处理它们要用码点视角的 `codePointAt`/`fromCodePoint`（或 `for..of`）；视觉上相同的字符可能是不同的组合序列（基础字符 + 变音记号），比较前必须 `str.normalize()` 归一化。
