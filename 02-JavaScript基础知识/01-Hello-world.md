# 01 Hello, world!

> 原文：`zh.javascript.info/1-js/02-first-steps/01-hello-world/article.md`

## 知识点

### 1. 运行环境
- 本部分讲的是 JS **语言本身**，浏览器只是一个方便的运行环境，教程会尽量少用浏览器特有 API（如 `alert`）。
- 服务器端（Node.js）直接用命令行运行：`node my.js`。

### 2. `<script>` 标签
- 可以放在 HTML 文档几乎任何位置，浏览器遇到就自动执行其中的代码。

```html
<script>
  alert('Hello, world!');
</script>
```

### 3. 已过时的写法（识别老代码用）
| 写法 | 说明 |
|---|---|
| `type="text/javascript"` | HTML4 要求，现在不需要；现代 HTML 中 `type` 被重新定义用于 ES 模块（`type="module"`） |
| `language="JavaScript"` | 已无意义，默认就是 JS |
| `<script><!-- ... //--></script>` | 为不支持 `<script>` 的远古浏览器隐藏代码，早已无用 |

### 4. 外部脚本
- 通过 `src` 引入：`<script src="/path/to/script.js"></script>`
- 路径可以是绝对路径、相对路径（`script.js` 等价于 `./script.js`）或完整 URL（CDN）。
- 引入多个脚本就写多个 `<script>` 标签。
- 好处：浏览器会**缓存**独立的脚本文件，多页面共用时只下载一次，省流量、加载快。

### 5. 重要限制
- **`src` 与内联代码不能共存**：设置了 `src`，标签内部的代码会被忽略。需要两者时拆成两个 `<script>` 标签。

```html
<script src="file.js"></script>
<script>
  alert(1);
</script>
```

## 与 Java 对比
- JS 没有 `main` 入口，脚本从上到下按顺序执行；浏览器中每个 `<script>` 就是一段被执行的代码。
- 不需要编译步骤，源码直接交给引擎（浏览器 / Node）解释执行。

## 一句话总结
用 `<script>` 把 JS 嵌入页面；外部脚本用 `src`；`type`/`language` 不再需要；`src` 和内联代码二选一。
