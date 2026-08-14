# double

## 题目简述

题目给出字符串 `Z2h7TnJpNHQ1aW5yeWFzN2FzME1UMF9OX3JOcFM3T31ldFJGNE5kNDBJ`。它先经过 Base64 编码，再经过栅栏密码（Rail Fence Cipher）换位。解题目标是识别这两层可逆变换，并按加密的逆序还原明文。

## 解题过程

字符串只含 Base64 常见字符且长度为 $4$ 的倍数，先进行 Base64 解码：

```python
import base64

data = b"Z2h7TnJpNHQ1aW5yeWFzN2FzME1UMF9OX3JOcFM3T31ldFJGNE5kNDBJ"
print(base64.b64decode(data).decode())
```

得到：

```text
gh{Nri4t5inryas7as0MT0_N_rNpS7O}etRF4Nd40I
```

这段文本保留了 `g`、`h`、花括号等 Flag 特征，但字符顺序明显被打乱，符合换位密码而不是再次编码。对其尝试常见栅栏数，使用三栏 Rail Fence 解密即可恢复：

```text
greyhats{7RaNsF0rM4Ti0N_4Nd_tr4N5p0Si7IOn}
```

## 方法总结

本题的关键是区分编码与密码：Base64 只改变表示形式，栅栏密码只改变字符位置。处理多层变换时应记录每一层的输入输出，并严格按加密顺序的逆序解码；中间结果中出现 Flag 格式特征，可用于判断方向是否正确。
