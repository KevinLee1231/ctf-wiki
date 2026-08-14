# notepad.exe

## 题目简述

附件伪装成记事本程序，静态特征表明它由 AutoIt 编译。程序启动系统记事本、写入干扰文本，并把 Flag 拆成静态字符串、注册表瞬时值、字符运算和 XOR 结果；其中两个注册表键写入后立刻删除。

目标是提取并反混淆 AutoIt 脚本，而不是分析原生 PE 的全部控制流。

## 解题过程

先在隔离环境中确认 AutoIt 特征，再用 AutoIt 脚本提取工具恢复 `.au3`。关键片段可分为：

```text
grey
hats{
L0L_        写入 HKCU\Software\SussyKeys1
wh0_        写入 HKCU\Software\SussyKeys2
```

注册表值随后被删除，因此动态分析时可用 Procmon 观察 `RegSetValue`，静态分析则直接读取恢复脚本中的 `RegWrite` 参数。

`BUTTERFLY()` 中的字符运算得到：

```python
part5 = "".join(map(chr, [115, 81 + 3, 79 - 30, 118 ^ 26, 108, 95]))
assert part5 == "sT1ll_"
```

数组第二项直接给出 `c0des_1n`。字符串 `@^jk/Vk@` 的每个字符再与 `0x1f` 异或：

```python
part7 = "".join(chr(ord(c) ^ 0x1f) for c in "@^jk/Vk@")
assert part7 == "_Aut0It_"
```

最后拼接固定结尾 `UwU}`，得到：

```text
greyhats{L0L_wh0_sT1ll_c0des_1n_Aut0It_UwU}
```

## 方法总结

- 核心技巧：识别 AutoIt 编译器并恢复脚本，联合静态字符串、注册表行为和简单字符运算重组 Flag。
- 识别信号：PE 内出现 AutoIt 运行时特征、程序自动控制窗口和键盘、注册表值短暂写入后删除。
- 复用要点：遇到脚本语言打包程序时先还原高级脚本；瞬时注册表数据既可静态读取 `RegWrite`，也可用行为监控交叉验证。
