# Oldschool

## 题目简述

`OldSchool.exe` 是一个经 UPX 压缩的 Windows PE。解包后可识别为 F2KO Bat To Exe 生成物：真正的批处理脚本被放在 RCDATA 资源中，先做逐字节取负、再用 RC4 加密，脚本正文还使用 BriefLZ 压缩。

## 解题过程

UPX 解包后的关键材料如下：

```text
seed offset       0x17f22, length 20
script resource   0x20e8c, length 0x0ea2
metadata resource 0x21d30, length 0x0020
```

20 字节种子为：

```text
ab3bb4215e2a346458f3ad6ceffab4167f3bd506
```

对其求 MD5，并使用小写十六进制文本作为 RC4 密钥：

```text
9af2d36735ade5eed589b9395233bdab
```

资源解密顺序不能颠倒：

```python
import hashlib

seed = image[0x17F22:0x17F22 + 20]
key = hashlib.md5(seed).hexdigest().encode()
encrypted = image[0x20E8C:0x20E8C + 0xEA2]
negated = bytes((-value) & 0xFF for value in encrypted)
decrypted = rc4(negated, key)
```

解密块前 4 字节给出脚本解压长度 `0x3714`，偏移 28 处是 BriefLZ 数据。解压出的批处理脚本要求 10 位十六进制 access key，把其 40 个 bit 重排成八组 5 bit，再用：

```text
ABCDEFGHJKLMNPRSTVWXYZ0123456789
```

编码。目标字符串是 `4S376BZH`。逆置换表如下：

```python
alphabet = "ABCDEFGHJKLMNPRSTVWXYZ0123456789"
target = "4S376BZH"
groups = [
    [32, 33, 34, 35, 36], [37, 38, 39, 8, 9],
    [10, 11, 12, 13, 14], [15, 0, 1, 2, 3],
    [4, 5, 6, 7, 27], [28, 29, 30, 31, 24],
    [25, 26, 16, 17, 18], [19, 20, 21, 22, 23],
]

bits = ["0"] * 40
for encoded, positions in zip(target, groups):
    for bit, position in zip(f"{alphabet.index(encoded):05b}", positions):
        bits[position] = bit

access_key = "".join(
    f"{int(''.join(bits[index:index + 4]), 2):X}"
    for index in range(0, 40, 4)
)
print(access_key)
```

得到 access key：

```text
DEF3A7C0D3
```

批处理脚本把目标码、常量 `27`、access key 和常量 `02` 重新组合，最终输出：

```text
UMDCTF-{4S3727DEF3A7C0D3026BZH}
```

其 SHA-256 与 README 中的 `19f81200ba2ce24aed7863eabf4c2d0c6dc45f6f082978ce71703957624e7bd2` 一致。

## 方法总结

这题的关键是逐层识别封装，而不是把解包后的 PE 当成最终程序：UPX、Bat To Exe 资源保护、RC4、BriefLZ 和批处理位重排各负责一层。每层都应检查头部、长度或已知格式，再进入下一层；最终 access key 来自可逆位排列，不需要暴力枚举 $16^{10}$。
