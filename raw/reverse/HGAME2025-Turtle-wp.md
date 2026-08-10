# Turtle

## 题目简述

附件是一个经过修改的 UPX 类壳保护的 Windows x64 程序。壳头和常见特征已被破坏，通用脱壳命令无法识别；脱壳后，程序先用标准 RC4 解出真正密钥，再用一个把异或改成加法的 RC4 变体恢复 flag。

## 解题过程

### 手工脱壳与 IAT 修复

由于 UPX 标识被修改，不能依赖自动脱壳。用调试器进入壳入口，按栈平衡/恢复特征跟踪解压 stub：在 stub 将原始栈恢复后继续运行，会跳转到原程序入口（官方题解中为 `0x4014E0`）。在该位置转储进程映像，再使用导入表重建工具扫描并修复 IAT，得到能够正常静态分析的 PE。

这一步的判断依据是控制流从大量解压、重定位操作转入常规函数序言和运行库调用，而不是仅凭某个固定地址套用“ESP 定律”。不同环境下装载地址可能变化，应以实际 OEP 行为为准。

### 恢复两阶段 RC4

脱壳后的主函数包含三个关键过程：

- `sub_401550`：标准 RC4 KSA，初始化 256 字节 S 盒；
- `sub_40163E`：标准 RC4 PRGA，以异或方式处理数据；
- `sub_40175A`：PRGA 状态更新不变，但将 `data[k] ^= rnd` 改成 `data[k] += rnd`。

第一阶段以字符串 `yekyek` 初始化 S 盒，再对 7 字节数组执行标准 RC4，得到第二阶段密钥：

```text
65 63 67 34 61 62 36  ->  ecg4ab6
```

第二阶段用该密钥重新初始化 S 盒，对 40 字节密文执行加法变体。下面的 Python 脚本复现了完整流程：

```python
def ksa(key: bytes) -> list[int]:
    s = list(range(256))
    j = 0
    for i in range(256):
        j = (j + s[i] + key[i % len(key)]) & 0xFF
        s[i], s[j] = s[j], s[i]
    return s


def prga(data: bytes, s: list[int], mode: str) -> bytes:
    out = bytearray(data)
    i = j = 0
    for k in range(len(out)):
        i = (i + 1) & 0xFF
        j = (j + s[i]) & 0xFF
        s[i], s[j] = s[j], s[i]
        rnd = s[(s[i] + s[j]) & 0xFF]
        if mode == "xor":
            out[k] ^= rnd
        elif mode == "add":
            out[k] = (out[k] + rnd) & 0xFF
        else:
            raise ValueError(mode)
    return bytes(out)


encrypted_key = bytes.fromhex("CD 8F 25 3D E1 51 4A")
ciphertext = bytes.fromhex(
    "F8 D5 62 CF 43 BA C2 23 15 4A 51 10 27 10 B1 CF "
    "C4 09 FE E3 9F 49 87 EA 59 C2 07 3B A9 11 C1 BC "
    "FD 4B 57 C4 7E D0 AA 0A"
)

key = prga(encrypted_key, ksa(b"yekyek"), "xor")
plain = prga(ciphertext, ksa(key), "add")

print(key.decode())
print(plain.decode())
```

输出为：

```text
ecg4ab6
hgame{Y0u'r3_re4l1y_g3t_0Ut_of_th3_upX!}
```

## 方法总结

- 核心技巧：在修改 UPX 特征导致自动工具失效时，动态定位 OEP、转储映像并修复 IAT；随后识别 KSA/PRGA 特征和被替换的数据组合运算。
- 识别信号：256 字节顺序初始化、`j = j + S[i] + key[...]`、交换 `S[i]`/`S[j]` 是 RC4 的强特征；PRGA 末尾不是异或时，应继续检查加减、位移等轻度魔改。
- 复用要点：手工脱壳的 OEP 应由运行时控制流验证，不能机械套地址。两阶段算法必须先恢复密钥再重置 S 盒，沿用第一阶段的 S 盒会得到错误结果。
