# Image Crash

## 题目简述

附件 `chall.png` 能通过 PNG 结构检查，却显示为严重错位的扫描线。IHDR 表明图像是 `730 × 405`、8 bit RGB；解压全部 IDAT 后，每行应由 1 字节滤波器类型和 $730\times3$ 字节像素差值组成。统计行首滤波器可见 50 行被写成类型 0，而相邻正常行主要使用类型 3、4，这符合“保存过程中滤波器字节损坏”的题意。

## 解题过程

PNG 的类型 2（Up）按上一行恢复当前行：

$$
\operatorname{Recon}(x)=\operatorname{Raw}(x)+\operatorname{Prior}(x)\pmod{256}
$$

将这 50 个异常的 0 改为 2 后，整幅图恢复连续。下面代码展示对解压扫描线的核心修复；重新压缩时需要同步重建 IDAT 的长度与 CRC：

```python
import struct
import zlib
from pathlib import Path

source = Path("chall.png").read_bytes()
offset = 8
chunks = []
compressed = bytearray()

while offset < len(source):
    length = struct.unpack(">I", source[offset:offset + 4])[0]
    kind = source[offset + 4:offset + 8]
    body = source[offset + 8:offset + 8 + length]
    chunks.append((kind, body))
    if kind == b"IDAT":
        compressed.extend(body)
    offset += 12 + length

raw = bytearray(zlib.decompress(compressed))
stride = 1 + 730 * 3
for row in range(405):
    filter_offset = row * stride
    if raw[filter_offset] == 0:
        raw[filter_offset] = 2

def chunk(kind, body):
    crc = zlib.crc32(kind + body) & 0xFFFFFFFF
    return struct.pack(">I", len(body)) + kind + body + struct.pack(">I", crc)

out = bytearray(b"\x89PNG\r\n\x1a\n")
inserted = False
for kind, body in chunks:
    if kind == b"IDAT":
        if not inserted:
            out += chunk(b"IDAT", zlib.compress(raw))
            inserted = True
        continue
    out += chunk(kind, body)

Path("recovered.png").write_bytes(out)
```

修复结果如下，底部字幕清晰显示 flag：

![修复错误 PNG 滤波器后，角色下方出现完整白色 flag 字幕](UMDCTF2017-image-crash-wp/recovered-flag.png)

```text
UMDCTF-{See_Thr0ugh_Th3_GL1Tch_4rT}
```

其 SHA-256 与 README 中的 `df53518cfae6607f44454a213cc1239bfaa7bba78be5786f453c5fa83c0f70e6` 一致。

## 方法总结

PNG 能解压并不代表扫描线参数正确。若图像呈现从某行开始持续传播的错位，应检查每行第一个滤波器字节，并利用上下行连续性判断错误类型。本题只修改滤波器元数据，没有猜测或改写像素正文；最终图像和官方摘要共同验证了修复。
