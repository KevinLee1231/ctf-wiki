# GreyCTF2022 - flappy-js

## 题目简述

题目是一款 JavaScript 小游戏。界面逻辑规定步数达到 `31337` 才显示 flag，但生成 flag 的函数和全部常量已经在客户端源码中，无需实际玩到阈值。

## 解题过程

在 `HUD.js` 中定位 `genFlag()`。函数先 Base64 解码常量，再把每个字节与 `0x55` 异或；随后截取到第一个 `=`，并连续执行六次 Base64 解码。

```python
from base64 import b64decode

v = b64decode(encoded)
v = bytes(x ^ 0x55 for x in v)
v = v[:v.index(b'=') + 1]
for _ in range(6):
    v = b64decode(v)
print(v.decode())
```

脚本输出内部字符串 `5uch_4_pr0_g4m3r`。显示代码会在外层拼上 `greyctf{` 与 `}`，所以最终为：

```text
greyctf{5uch_4_pr0_g4m3r}
```

## 方法总结

客户端游戏的得分门槛通常只是显示条件，不是秘密保护。应先全文搜索 flag 生成、解码和阈值判断，再离线复现最小数据流。多层 Base64 要保留每层的字节语义与填充，不能把中间结果随意按文本清洗。
