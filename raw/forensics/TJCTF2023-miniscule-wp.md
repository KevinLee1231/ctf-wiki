# miniscule

## 题目简述

附件具有正确 PNG 签名和常见的 `IHDR`、`IDAT`、`IEND` 结构，却无法被普通查看器打开，体积也比原图异常地小。源码表明它把 PNG 的压缩方法字段写成 1，并用 Zstandard 压缩扫描线；PNG 规范只允许压缩方法 0，也就是 zlib/DEFLATE。

## 解题过程

不能只把 `IHDR` 的压缩方法字节从 1 改成 0，因为 `IDAT` 载荷仍是 Zstandard 数据。完整修复过程是：

1. 解析三个 chunk，取出 `IHDR` 与 `IDAT` 数据；
2. 把 `IHDR` 压缩方法改成 0，并重算该 chunk 的 CRC32；
3. 用 Zstandard 解压 `IDAT`，再用 zlib 压缩；
4. 重建 `IDAT` 的长度、类型、数据与 CRC32，最后拼回 `IEND`。

```python
import zlib
import zstandard

raw = open("miniscule.png", "rb").read()
signature, chunks = raw[:8], raw[8:]
body, iend = chunks[:-12], chunks[-12:]
ihdr, idat = body[:25], body[25:]

ihdr = ihdr[:18] + b"\x00" + ihdr[19:-4]
ihdr += zlib.crc32(ihdr[4:]).to_bytes(4, "big")

scanlines = zstandard.decompress(idat[8:-4])
compressed = zlib.compress(scanlines, level=9)
idat = len(compressed).to_bytes(4, "big") + b"IDAT" + compressed
idat += zlib.crc32(idat[4:]).to_bytes(4, "big")

open("decoded.png", "wb").write(signature + ihdr + idat + iend)
```

修复后的图片能够正常渲染，底部直接显示 flag：

![修复 PNG 压缩方法和 IDAT 后显示宝可梦图案及底部 flag](TJCTF2023-miniscule-wp/repaired-png-flag.png)

```text
tjctf{zlib_compression_bad_9c8b342}
```

## 方法总结

- 文件有正确 magic 不代表内部字段有效；应继续检查 chunk 长度、类型、规范枚举值与 CRC。
- 修改容器声明时必须同步转换实际载荷并重算校验值，单改文件头通常只会制造另一种损坏。
- 保留最终修复图有助于证明像素数据完整恢复；结构和命令则应直接写入正文，避免让图片承担代码说明。
