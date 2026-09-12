# 12 JSON 方法，toJSON

> 原文：`zh.javascript.info/1-js/05-data-types/12-json/article.md`

## 知识点

### 1. 两个核心方法
- `JSON.stringify(value)` —— 对象 → JSON 字符串（序列化/编组）
- `JSON.parse(str)` —— JSON 字符串 → 对象（反序列化）

```js
let student = {
  name: 'John', age: 30, isAdmin: false,
  courses: ['html', 'css', 'js'], spouse: null
};
let json = JSON.stringify(student);
// {"name":"John","age":30,"isAdmin":false,"courses":["html","css","js"],"spouse":null}
```

### 2. JSON 格式的严格规则（与对象字面量的区别）
- 字符串**必须双引号**（没有单引号/反引号）。
- **属性名也必须双引号**：`age:30` → `"age":30`。
- 不支持注释、`new` 调用、尾随逗号。
- 支持的类型：object、array、string、number、boolean、`null`。
- 严格是为了让解析算法简单可靠快速。（另有独立库 JSON5 支持宽松格式，不在语言规范内。）

### 3. 会被跳过的属性
JSON 是语言无关的纯数据规范，以下 JS 特有内容被 `stringify` **忽略**：
- 函数属性（方法）
- Symbol 类型的键和值
- 值为 `undefined` 的属性

```js
let user = {
  sayHi() {},            // 忽略
  [Symbol("id")]: 123,   // 忽略
  something: undefined   // 忽略
};
JSON.stringify(user); // "{}"
```

### 4. 嵌套对象自动转换；循环引用报错
```js
JSON.stringify(meetup); // 嵌套对象、数组都自动递归转换

meetup.place = room;
room.occupiedBy = meetup;
JSON.stringify(meetup); // Error: Converting circular structure to JSON
```

### 5. replacer：排除和转换（第二参数）
完整语法：`JSON.stringify(value[, replacer, space])`

**用法一：属性名数组（白名单）**——只序列化列出的属性：
```js
JSON.stringify(meetup, ['title', 'participants']);
// {"title":"Conference","participants":[{},{}]}
// ⚠️ 白名单应用于整个结构的所有层级，participants 里的 name 不在列表所以丢了

JSON.stringify(meetup, ['title', 'participants', 'place', 'name', 'number']);
// 除 occupiedBy 外都序列化了 —— 用白名单绕开循环引用
```

**用法二：函数 `function(key, value)`**——每个键值对都会调用，**递归**应用于嵌套对象和数组项：
- 返回替换后的值；返回 `undefined` 表示跳过。
- 函数里的 `this` 是包含当前属性的对象。

```js
JSON.stringify(meetup, function replacer(key, value) {
  return (key == 'occupiedBy') ? undefined : value;
});
```
- 第一次调用特殊：键为空字符串 `""`，值是整个目标对象（包装对象 `{"": meetup}`），给 replacer 一个处理整体的机会。

### 6. space：格式化输出（第三参数）
```js
JSON.stringify(user, null, 2);
/*
{
  "name": "John",
  "age": 25,
  "roles": {
    "isAdmin": false,
    "isEditor": true
  }
}
*/
```
- 数字 = 缩进空格数；也可以是字符串（用作缩进符）。
- 仅用于日志/美化，网络传输不需要。

### 7. 自定义 `toJSON`
对象提供 `toJSON()` 方法时，`JSON.stringify` 自动调用它（类似 `toString` 之于字符串转换）：

```js
let room = {
  number: 23,
  toJSON() { return this.number; }
};

JSON.stringify(room);   // 23
JSON.stringify({title: "Conference", room}); // {"title":"Conference","room":23}
```
- 直接调用和嵌套时都生效。
- `Date` 有内建 `toJSON`，所以日期序列化成 `"2017-01-01T00:00:00.000Z"` 这种 ISO 字符串。

### 8. `JSON.parse` 与 reviver
```js
let numbers = JSON.parse("[0, 1, 2, 3]");
let user = JSON.parse('{"name":"John","age":35,"friends":[0,1,2,3]}');
user.friends[1]; // 1
```

**经典问题：Date 复活**。反序列化后日期字符串不会自动变回 Date：

```js
let str = '{"title":"Conference","date":"2017-11-30T12:00:00.000Z"}';
let meetup = JSON.parse(str);
meetup.date.getDate(); // Error！date 是字符串
```

**reviver 函数**（第二参数）对每个 `(key, value)` 调用，可转换值：

```js
let meetup = JSON.parse(str, function(key, value) {
  if (key == 'date') return new Date(value);
  return value;
});
meetup.date.getDate(); // ✓ 嵌套对象里的 date 也生效
```

## 与 Java 对比
| Java（Jackson/Gson） | JS | 说明 |
|---|---|---|
| `ObjectMapper.writeValueAsString(obj)` | `JSON.stringify(obj)` | JS 内建，无需库 |
| `ObjectMapper.readValue(str, X.class)` | `JSON.parse(str)` | JS 不需要目标类，动态类型 |
| `@JsonIgnore` / 序列化过滤 | replacer（数组白名单或函数） | replacer 函数 ≈ 自定义 Serializer |
| `@JsonProperty` 自定义 | `toJSON()` 方法 | 类上挂方法即自定义序列化 |
| 循环引用 Jackson 也报错（可用 `@JsonIdentityInfo`） | 循环引用直接抛错，用 replacer 排除 | 一致的痛点 |
| `writeValueAsString` + `SerializationFeature.INDENT_OUTPUT` | `stringify(obj, null, 2)` | 美化输出 |
| 反序列化日期要配 `JavaTimeModule` | reviver 手动 `new Date(value)` | JS 完全靠 reviver 回调 |
| JSON 支持类型 | object/array/string/number/boolean/null | 一致；函数/symbol/undefined 被跳过 |

## 一句话总结
`JSON.stringify` 序列化、`JSON.parse` 反序列化；JSON 格式严格（双引号、键加引号、无注释），函数/Symbol/undefined 被跳过，循环引用会报错；`stringify` 的 replacer（白名单数组或 `(key,value)` 函数）控制排除与转换、space 控制美化缩进；对象可用 `toJSON()` 自定义序列化（Date 内建就有）；反序列化用 reviver 把日期字符串等特殊值复活成对应类型。
