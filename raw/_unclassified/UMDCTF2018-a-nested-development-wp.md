# UMDCTF 2018 - A Nested Development

## 题目简述

题目只给出一个 41 MB 文件 `Nested` 和提示 “Clean the nest”。文件被数百层压缩、归档和文本编码反复包裹，需要按当前层的真实格式逐层解开。

## 解题过程

首层魔数为 `fd 37 7a 58 5a 00`，即 XZ。解压后得到 uuencode 文本，随后又出现 gzip、tar、bzip2、ZIP 和 Base64。适合写一个每轮重新识别格式的解包器：

```python
def unpack_once(data):
    if data.startswith(b"\xfd7zXZ\x00"):
        return "xz", lzma.decompress(data)
    if data.startswith(b"\x1f\x8b"):
        return "gzip", gzip.decompress(data)
    if data.startswith(b"BZh"):
        return "bzip2", bz2.decompress(data)
    if data.startswith(b"begin "):
        return "uuencode", decode_uu(data)
    if data.startswith(b"PK\x03\x04"):
        return "zip", extract_only_member(data)
    if is_tar(data):
        return "tar", extract_only_member(data)
    if is_strict_base64(data):
        return "base64", base64.b64decode(data)
    return None
```

每个归档层都只取唯一的普通文件，并记录格式、输入长度和输出长度。前几层为：

```text
0000 xz       41260188 -> 54639461
0001 uuencode 54639461 -> 39657654
0002 gzip     39657654 -> 39651586
0005 tar      52346880 -> 52342423
```

后段逐渐变成连续 Base64。第 365 层把 68 字节解码为最终 49 字节：

```text
UMDCTF-{n0w_yoU_caN_GO_reAd_SomE_Phr4cK_ARtiCLes}
```

总计成功解开 366 层。

## 方法总结

多层套娃题不能假设格式按固定周期出现。稳妥做法是每轮根据魔数和严格语法重新识别，并要求 ZIP/TAR 只有一个目标文件、Base64 必须通过完整校验，同时记录每层长度以便定位误判。
