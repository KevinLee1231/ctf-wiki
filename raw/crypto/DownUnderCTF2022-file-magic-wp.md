# DownUnderCTF 2022 file-magic Writeup

## 题目简述

服务要求提交一个 16 字节 IV 和不足 1337 字节的数据。原始数据必须能被 Pillow 识别为 $13\times37$ JPEG，像素 `(7,7)` 必须是 `(7,7,7)`；同一数据经已知密钥 `downunderctf2022` 做 AES-CBC 解密后又必须以 ELF 魔数开头，随后会被写入临时文件并执行。目标是构造“JPEG 密文 / ELF 明文”双重格式载荷。

## 解题过程

先编写一个尽量小的 64 位 ELF。官方载荷使用 `open("flag.txt")` 和 `sendfile(1, fd, ...)` 把 flag 输出到标准输出；ELF 尾部可容忍额外垃圾，因此不必让整个解密结果都属于程序映像。

AES-CBC 第一块满足
$P_0=D_K(C_0)\mathbin{\mathrm{xor}}IV$，等价地
$IV=D_K(C_0)\mathbin{\mathrm{xor}}P_0$。
密钥已知且 IV 可控，所以可以先固定一个合法 JPEG 开头作为 $C_0$，再由目标 ELF 头计算 IV。

选择的 16 字节首块为：

```text
ff d8 ff ff fe LL LL 78 78 78 78 78 78 78 78 78
```

`ff d8` 是 JPEG SOI，`ff fe` 是注释段标记，中间额外的 `ff` 可作为填充；`LL LL` 是大端注释长度。后面的 AES 密文块会被 JPEG 解析器当作注释内容，从而不影响后续真正的 JPEG 图像段。

题目还要求 IV 中出现 `DUCTF`。ELF `e_ident` 的第 8 到 12 字节是可自由调整的填充区。令 `q = AES_ECB_decrypt(C0)`，把 ELF 首块的这五个字节设为 `q[8:13] XOR b'DUCTF'`，再计算 `IV = q XOR P0`，即可保证 IV 的同一位置恰为 `DUCTF`，同时 CBC 解密仍得到修改后的合法 ELF 头。

核心构造如下：

```python
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad
from Crypto.Util.strxor import strxor
from io import BytesIO
from PIL import Image

key = b'downunderctf2022'
elf = bytearray(open('catflag', 'rb').read())

image = Image.new('RGB', (13, 37))
image.putpixel((6, 7), (69, 69, 69))
buf = BytesIO()
image.save(buf, format='JPEG')
jpeg = buf.getvalue()

comment_len = len(elf) + 9 + ((len(elf) % 16) or 16)
c0 = b'\xff\xd8\xff\xff\xfe' + comment_len.to_bytes(2, 'big') + b'x' * 9
q = AES.new(key, AES.MODE_ECB).decrypt(c0)

elf[8:13] = strxor(q[8:13], b'DUCTF')
iv = strxor(q, bytes(elf[:16]))
assert b'DUCTF' in iv

cipher = AES.new(key, AES.MODE_CBC, iv).encrypt(pad(bytes(elf), 16))
payload = pad(cipher + jpeg[2:], 16)
```

这里把正常 JPEG 去掉 SOI 后追加到加密 ELF 之后；首块已经提供 SOI 和注释头，注释长度会跨过中间密文，随后 Pillow 继续解析追加的 JPEG 段。题目给定的像素微调使有损压缩后的 `(7,7)` 正好为 `(7,7,7)`。

提交 `iv.hex()` 和 `payload.hex()` 后，服务执行解密得到的 ELF，输出：

```text
DUCTF{y0u_4r3_4_f1l3_m4g1c14n_ab8a8133fef871becaf1}
```

## 方法总结

这是基于 CBC 首块可控关系构造的格式多态。应把两套解析器分开考虑：JPEG 只需把中间密文吞进注释段并在后部找到有效图像，ELF 只需首部和映射段有效且可忽略尾部垃圾。利用 ELF 头的非关键填充字节还能反向满足 IV 必含 `DUCTF` 的额外约束；盲目搜索同时合法的随机文件远不如直接利用 $P_0=D_K(C_0)\mathbin{\mathrm{xor}}IV$ 设计。
