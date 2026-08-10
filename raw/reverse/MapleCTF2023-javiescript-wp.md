# javiescript

## 题目简述

题目是一段刻意利用 JavaScript 隐式转换的校验逻辑。查询参数会被解析并赋给对象 `honk`，程序一方面要求 `Object.keys(honk).length === 0`，另一方面又读取 `honk.one`、`honk.two` 和 `honk.three` 拼接 flag。直接提交三个自有属性会违反键数量检查。

## 解题过程

向普通对象的 `__proto__` 属性赋值会触发原型设置器。把三个字段放在新原型上，它们能被属性查找读到，却不会出现在 `Object.keys` 返回的自有可枚举键中。原始参数值为：

```json
{
  "__proto__": {
    "one": "NaN",
    "two": [null],
    "three": "_are_a_mId_FruiT}"
  }
}
```

实际请求中应将整个 JSON 作为 `__proto__` 参数值进行 URL 编码，避免 `&`、引号或花括号被查询解析器提前拆分。

各字段是针对后续强制类型转换精确选择的。对象继承的 `toString()` 返回 `[object Object]`，指定下标取到字母 `b`；`eval("NaN")` 得到数值 NaN，NaN 的乘法与字符串比较分支产生 `aN`；`[null].toString()` 为空串但数组长度为 1，使循环补出 `annnnas`。最后附加 `three` 的固定后缀，完整结果为：

```text
maple{baNannnnas_are_a_mId_FruiT}
```

## 方法总结

JavaScript 对象的“可读取属性”和“自有可枚举属性”不是同一集合。安全校验若用 `Object.keys` 判断空对象，却随后通过普通属性访问取值，就可能被原型属性绕过。审计此类题时应逐个写出隐式转换结果，包括 `toString`、数组长度、NaN 比较和索引字符，不要凭直觉猜最终字符串。
