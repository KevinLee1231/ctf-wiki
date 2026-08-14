# ColoursOfTheRainbow

## 题目简述

题目复用上一题 `LetMeIn.apk`，提示从反编译后的 Android 资源中寻找颜色。决定性步骤不是移动平台漏洞，而是把伪装成颜色值的十六进制数据按彩虹顺序重新排列并解码。

## 解题过程

在反编译目录的 `resources/res/values/colors.xml` 中，除正常主题颜色外还有七个名称异常简短的条目。按提示 `roygbiv`（red、orange、yellow、green、blue、indigo、violet）排列如下：

| 名称 | 十六进制值 | ASCII |
| --- | --- | --- |
| `r` | `67726579` | `grey` |
| `o` | `68617473` | `hats` |
| `y` | `7b536576` | `{Sev` |
| `g` | `336e5f52` | `3n_R` |
| `b` | `40696e62` | `@inb` |
| `i` | `30777337` | `0ws7` |
| `v` | `3737377d` | `777}` |

可以直接按顺序拼接后解码：

```python
values = [
    "67726579", "68617473", "7b536576", "336e5f52",
    "40696e62", "30777337", "3737377d",
]
print(b"".join(bytes.fromhex(x) for x in values).decode())
```

输出为：

```text
greyhats{Sev3n_R@inb0ws7777}
```

## 方法总结

资源名提供顺序，资源值提供载荷，两者缺一不可。遇到看似 ARGB 的 8 位十六进制值时，可以检查每两个十六进制字符是否落在可打印 ASCII 范围；若单块只形成短字符串，还要结合命名、题目提示或自然顺序重组，而不能按 XML 出现顺序盲目拼接。
