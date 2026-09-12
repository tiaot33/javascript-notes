# 07 "new Function" 语法

> 原文：`zh.javascript.info/1-js/06-advanced-functions/07-new-function/article.md`
> 极少使用，但它是**闭包规则的唯一例外**，特定场景（动态代码）只能用它。

## 知识点

### 1. 语法
```js
let func = new Function([arg1, arg2, ...argN], functionBody);

let sum = new Function('a', 'b', 'return a + b');
sum(1, 2); // 3

let sayHi = new Function('alert("Hello")'); // 无参数，只有函数体
sayHi(); // Hello
```

- 参数都是**字符串**，函数体也是字符串，在**运行时**才被编译成函数。
- 历史原因支持逗号分隔参数：`new Function('a,b', 'return a+b')` 与上面等价。
- 与函数声明/表达式的本质区别：**函数代码可以来自任何来源**，比如从服务器获取字符串再执行：

```js
let str = ...; // 动态接收的代码
let func = new Function(str);
func();
```

### 2. 闭包例外：`[[Environment]]` 指向全局
普通函数通过 `[[Environment]]` 记住创建地的词法环境；`new Function` 创建的函数 `[[Environment]]` **指向全局环境**，因此**访问不到外部变量，只能访问全局变量**：

```js
function getFunc() {
  let value = "test";
  let func = new Function('alert(value)');
  return func;
}
getFunc()(); // ❌ Error: value is not defined

// 普通函数对比：
function getFunc2() {
  let value = "test";
  return function() { alert(value); };
}
getFunc2()(); // ✓ "test"
```

### 3. 为什么这样设计（其实是好事）
- 若 `new Function` 能访问外部变量，代码经**压缩器（minifier）**缩短局部变量名后（`userName` → `a`），运行时才创建的函数就找不到了。
- 架构上，依赖隐式外部变量也更容易出错。
- 结论：**给 `new Function` 的函数传数据，必须显式走参数**。

### 4. 使用场景
- 复杂的 Web 应用中动态编译模板函数、执行服务端下发的代码等极特殊场景。
- 日常代码不要用（和 `eval` 一样，有安全和调试成本）。

## 与 Java 对比
| Java | JS | 说明 |
|---|---|---|
| 运行时动态生成类：字节码生成（ASM/CGLIB）、`javax.tools.JavaCompiler`、脚本引擎 JSR-223 | `new Function('a','b','return a+b')` | JS 一行字符串造函数；Java 要重得多 |
| 动态生成的类无法访问调用方局部变量 | `new Function` 只能访问全局变量 | 限制类似，都是为了隔离 |
| 无直接对应（`eval` 已废弃 Nashorn） | 字符串即代码 | 动态语言特性，Java 静态编译世界观里没有等价物 |

## 一句话总结
`new Function('a', 'b', 'return a + b')` 在运行时用字符串造函数，适合"代码来源动态"的极特殊场景（模板编译、服务端下发）；它的 `[[Environment]]` **指向全局环境**——是 JS 中唯一不是天生闭包的函数形式，访问不到外部局部变量（这既避免了压缩器改名带来的隐患，也强制数据显式通过参数传递）。
