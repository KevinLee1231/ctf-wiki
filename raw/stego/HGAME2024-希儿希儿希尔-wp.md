# 希儿希儿希尔

## 题目简述

附件 `secret.png` 是一条多层隐写链：PNG 的宽高字段被破坏；修复后，文件尾部还拼接了包含 Hill 密文的 ZIP；同时，图片 RGB 通道最低有效位中隐藏了 Hill 密钥。只有依次完成结构修复、附加文件提取、LSB 提取和 Hill 解密，才能恢复 flag。

## 解题过程

### 修复 PNG 尺寸

Windows 图片查看器可能把损坏图片显示成噪声，部分 Linux/macOS 查看器会直接拒绝。检查 PNG 的 `IHDR` 后，将宽、高恢复为题解验证过的 `1444 × 1444`。这两个值位于 `IHDR` 的宽高字段中并采用大端序；手工修改后还应重新计算对应 CRC，避免严格解析器继续报错。

修复成功后，`binwalk` 能同时识别正常 PNG 数据和文件尾部的 ZIP：

```bash
binwalk secret.png
binwalk -e secret.png
```

提取出的 ZIP 中有 `secret.txt`，内容是：

```text
CVOCRJGMKLDJGBQIUIVXHEYLPNWR
```

### 从 RGB 最低位恢复密钥

按像素顺序依次读取 R、G、B 三个通道的最低位，每 8 bit 还原一个字节：

```python
from PIL import Image


def extract_lsb(path: str, byte_count: int) -> bytes:
    image = Image.open(path).convert("RGB")
    bits = []

    for pixel in image.getdata():
        for channel in pixel:
            bits.append(str(channel & 1))
            if len(bits) == byte_count * 8:
                return bytes(
                    int("".join(bits[index:index + 8]), 2)
                    for index in range(0, len(bits), 8)
                )

    raise ValueError("图片容量不足")


print(extract_lsb("secret.png", 16))
```

输出为：

```text
KEY:[[8 7][3 8]]
```

因此 Hill 密钥矩阵为：

$$
K=
\begin{pmatrix}
8 & 7\\
3 & 8
\end{pmatrix}.
$$

其行列式为 $43\equiv17\pmod{26}$，而 $\gcd(17,26)=1$，所以矩阵在模 26 下可逆。逆矩阵是：

$$
K^{-1}\equiv
\begin{pmatrix}
2 & 21\\
9 & 2
\end{pmatrix}
\pmod{26}.
$$

### Hill 解密

按两个字母一组映射到 $0\ldots25$，再左乘 $K^{-1}$：

```python
ciphertext = "CVOCRJGMKLDJGBQIUIVXHEYLPNWR"
inverse_key = ((2, 21), (9, 2))

numbers = [ord(char) - ord("A") for char in ciphertext]
plaintext = []

for index in range(0, len(numbers), 2):
    left, right = numbers[index:index + 2]
    plaintext.append((inverse_key[0][0] * left + inverse_key[0][1] * right) % 26)
    plaintext.append((inverse_key[1][0] * left + inverse_key[1][1] * right) % 26)

print("".join(chr(value + ord("A")) for value in plaintext))
```

得到：

```text
DISAPPEARINTHESEAOFBUTTERFLY
```

还原为题目 flag 格式即 `hgame{DISAPPEARINTHESEAOFBUTTERFLY}`。

## 方法总结

- 核心技巧：修复 PNG 结构、提取尾部 ZIP、读取 RGB LSB，再进行模 26 Hill 矩阵解密。
- 识别信号：图片在不同平台表现不一致、`binwalk` 发现尾随压缩包、密文为偶数长度的大写字母串、题目名暗示 Hill。
- 复用要点：多层隐写要按载体结构从外到内处理；Hill 密钥必须在模 26 下可逆，并要确认采用列向量还是行向量约定。
