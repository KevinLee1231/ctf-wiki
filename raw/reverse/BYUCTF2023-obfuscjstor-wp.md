# BYUCTF 2023 - obfuscJStor

## 题目简述

附件是一段典型 JavaScript Obfuscator 风格代码：字符串被十六进制转义后放入数组，启动 IIFE 循环旋转数组，再通过减去固定偏移的索引函数取值。

## 解题过程

先把代码放入隔离的 Node/浏览器调试环境，在旋转 IIFE 完成后打印 `_0x12de()` 返回的数组，或直接在 `hi()` 中把每个 `_0x398601(0x...)` 替换成实际字符串。无需猜测控制流；数组中已经包含：

```text
byuctf
f{one
_of_th
ese_days_
imma_
make_
a_tool_to_
deobfuscate_this}
```

按 `console.log` 中的调用顺序拼接，恢复：

```text
byuctf{one_of_these_days_imma_make_a_tool_to_deobfuscate_this}
```

源码还用 `document.domain` 条件控制输出；静态替换或在调试器中直接调用拼接表达式，可以避开域名环境依赖。

## 方法总结

这类混淆的第一目标是还原字符串表与索引映射，而不是逐条理解无意义算术。数组旋转完成后建立“索引到明文”的字典，控制流会迅速变得可读。
