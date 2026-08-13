# GreyCTF2024 No More Tones WP

## 题目简述

附件是一个简化 OFDM 波形。每个字符占 40 个采样点，其中前 8 点是循环前缀，后 32 点是实数 IFFT 符号；频域第 4 到第 11 号 bin 承载 7 个数据位和 1 个奇偶校验位，并对相邻字符做差分编码。

## 解题过程

生成器在有效数据前填充 1234 个零、结尾填充 4321 个零，因此先精确裁掉两端。每 40 点取一块，丢弃前 8 点循环前缀，对剩余 32 点做 `rfft`。第 4 到第 11 个频点实部的正负分别代表 1 和 0。

差分状态初始为全零。生成器令当前星座符号等于“本字符位异或上一字符位”，所以解码时再与累计状态异或即可恢复当前字符位：

```python
import numpy as np
from scipy.io import wavfile

fs, data = wavfile.read("nomoretones.flac")
data = data[1234:-4321]
state = np.zeros(8, dtype=np.uint8)
out = []

for pos in range(0, len(data), 40):
    symbol = data[pos:pos + 40][-32:]
    signs = (np.real(np.fft.rfft(symbol))[4:12] > 0).astype(np.uint8)
    state ^= signs
    value = sum(int(state[i]) << i for i in range(7))
    out.append(chr(value))

print("".join(out))
```

数据位采用低位在前的顺序，即字符值为 $\sum_{i=0}^{6}b_i2^i$。解码文本中多次重复出现：

```text
grey{0FDM_Modulat10n_M4g1c_4h398trh38rh9438}
```

## 方法总结

OFDM 解码的关键顺序是：去静音、按符号长度分块、移除循环前缀、FFT 取子载波、撤销差分、按位序组字节。若直接把每个符号的频域符号当明文位，第二个字符起就会错误；必须维护上一字符的差分状态。
