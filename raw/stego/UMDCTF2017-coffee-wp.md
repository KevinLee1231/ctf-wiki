# Coffee

## 题目简述

`coffee.png` 是一张 `480 × 352` 的 24 bit RGB PNG。文件没有附加数据或异常辅助块，标准 LSB 扫描也没有直接得到文本。README 明确说明咖啡主题与解法无关，真正载荷藏在像素通道的单个位平面中。

## 解题过程

按行优先展开 RGB 像素，分别检查通道与 bit 位。将最低有效位记为 bit 0 时，红色通道 bit 2，也就是掩码 `0x04`，按 MSB-first 打包后在偏移 9 出现合法 gzip 头：

```text
1f 8b 08 08 1e b7 c6 58 ...
```

头部中的压缩方法、原文件名标志和 2017 年时间戳也都合理，足以排除随机魔数。提取与解压代码如下：

```python
import zlib

import numpy as np
from PIL import Image

pixels = np.asarray(Image.open("coffee.png").convert("RGB"), dtype=np.uint8)
red = pixels[:, :, 0].reshape(-1)
bits = ((red >> 2) & 1).astype(np.uint8)
blob = np.packbits(bits, bitorder="big").tobytes()

offset = blob.index(b"\x1f\x8b\x08")
decoder = zlib.decompressobj(wbits=31)
payload = decoder.decompress(blob[offset:]) + decoder.flush()
print(payload.decode())
```

gzip 成员结束后还有其他打包位形成的尾随数据，所以使用 `decompressobj` 在第一个成员结束处停止。输出：

```text
UMDCTF-{C0ff33_Is_tH3_L1f3blood_th4t_FU3Ls_ThE_DreamS_OF_CH4Mp10nS}
```

其 SHA-256 正好是 README 中的 `00cde75ea65c33100e4fb244c0fab9ef560feb08f6a130fd25882e928b882586`。

## 方法总结

位平面载荷不一定放在整体 RGB 的最低位，也不一定从字节边界 0 开始。系统检查时应枚举通道、位号、像素遍历和 bit 打包顺序，并用 gzip、PNG、ZIP 等完整头部字段验证候选。本题由 gzip 结构和官方 SHA-256 构成了完整证据链。
