# EasyVT

## 题目简述

题目由 `EasyVT.sys` 内核驱动和 `Guest.exe` 用户态程序组成，模拟 Intel VT-x 虚拟化。驱动建立 VMX 环境，Guest 通过 VMX 指令触发 VM exit，驱动 switch-case handler 负责初始化数据、执行加密变换并最终校验。

核心算法是 TEA 变体和 RC4 的组合。由于动态运行 VT 环境成本较高，解法以静态分析驱动 handler、提取比较数组和密钥为主。

## 解题过程

### 关键观察

驱动中的 VM exit handler 分三类：

```text
vmxon ~ vmread       数据初始化
vmcall ~ vmresume    加密变换
vmxoff               最终校验
```

`DAT_00404010` 处存放 10 个 `uint32`，使用前 8 个作为 4 组 TEA 输入：

```text
5C073994 0D805CB3 87DDA586 0317FB8E
6520EF29 5A4987AF EB2DC2A4 38CF470E
```

TEA 参数：

```text
rounds = 32
delta  = 0xC95D6ABF
sum0   = 0x20000000 - delta * 32
key    = [0x00102030, 0x40506070, 0x8090A0B0, 0xC0D0E0F0]
```

RC4 key 是 ASCII：

```text
04e52c7e31022b0b
```

### 求解步骤

对每组数据先按驱动中的 TEA 变体解密，再按 `(v1, v0)` 小端顺序拼接，最后用 RC4 解密。

TEA 核心：

```python
def tea_decrypt(v0, v1):
    key = [0x00102030, 0x40506070, 0x8090A0B0, 0xC0D0E0F0]
    delta = 0xC95D6ABF
    s = (0x20000000 - delta * 32) & 0xffffffff
    l, r = v0, v1
    for _ in range(32):
        s = (s + delta) & 0xffffffff
        r = (r - (((l >> 5) + key[0]) ^ (l + s) ^ ((l << 4) + key[2]))) & 0xffffffff
        l = (l + (((r >> 5) + key[3]) ^ (r + s) ^ ((r << 4) + key[1]))) & 0xffffffff
    return l, r
```

解出 4 组中间值后经 RC4 得到：

```text
DASCTF{81920c3758be43705ba154bb8f599846}
```

## 方法总结

- 核心技巧：从 VM exit handler 中还原实际加密流程，而不是尝试完整复现 VT 运行环境。
- 识别信号：驱动题中出现 VMX 指令、guest/host 调度和 handler switch 时，要把 VM 指令当作字节码调度入口。
- 复用要点：TEA 变体要逐项确认轮数、delta、sum 初值、加减方向和 key 顺序；组合加密题要保留中间打包顺序，否则 RC4 输入会错位。
