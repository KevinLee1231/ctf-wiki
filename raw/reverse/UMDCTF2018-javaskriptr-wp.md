# UMDCTF 2018 - JavaSkriptr

## 题目简述

题目把 flag 拆在前端 JavaScript 数组中：前 8 项是已知前缀 `UMDCTF-{`，后续项目是数字。核心不是 Web 权限绕过，而是还原客户端脚本中的循环异或，因此归入逆向。

## 解题过程

数组的关键部分为：

```javascript
var a = [
  "U", "M", "D", "C", "T", "F", "-", "{",
  51, 31, 43, 45, 32, 25, 30, 21,
  49, 18, 55, 112, 55, 51, 95, 74,
  33, 52, 57
];
```

循环用后半段数字与前缀的 8 个字符循环异或。页面中的生产版脚本计算了临时变量却没有把它追加到输出字符串，但算法仍可从表达式直接恢复：

```python
prefix = "UMDCTF-{"
values = [
    51, 31, 43, 45, 32, 25, 30, 21,
    49, 18, 55, 112, 55, 51, 95, 74,
    33, 52, 57,
]

tail = "".join(
    chr(value ^ ord(prefix[index % len(prefix)]))
    for index, value in enumerate(values)
)
print(prefix + tail)
```

输出为：

```text
UMDCTF-{fRont_3nd_s3cur1ty}
```

其 SHA-1 为：

```text
f040ccdd740badc1de428cc2a1de1041a4c56588
```

与 README 的官方摘要一致。

## 方法总结

前端源码对访问者完全可见，混淆或删掉显示语句并不能保护秘密。遇到已知前缀加数字数组时，可利用前缀既作为密钥又作为格式校验，直接复现运算并用官方摘要确认结果。
