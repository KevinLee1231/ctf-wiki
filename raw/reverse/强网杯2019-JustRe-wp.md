# JustRe

## 题目简述

题目附件是 `task_JustRe6.tar.gz`，其中包含 `JustRe.exe`。现有解题脚本显示，程序把 flag 分成两段验证：前半段通过 DWORD 级别的加法和低位 XOR 关系恢复，后半段通过 3DES-ECB 解密固定密文恢复。

最终 flag 由两段拼接而成：

```text
flag{13242238160dcc509a6f75849b}
```

## 解题过程

### 关键观察

脚本读取 `JustRe.exe` 中两类数据：

```text
target_dwords : 文件偏移 0x1f48，24 个 DWORD
initial_bytes : RVA 0x405018 对应的 96 字节初始表
```

第一段校验的形态可整理为：

```python
add_val = b * 0x01010101
computed = (initial_dword + add_val) ^ ((j + iVar5) & 0x1f)
```

逐个 `j` 检查 `computed` 与目标低字节的关系。第二段是固定 3DES-ECB：

```text
ciphertext = 507CA9E68709CEFA20D50DCF90BB976C9090F6B07BA6A4E8
key        = AFSAFCEDYCXCXACNDFKDCQXC
```

脚本对 3DES key 字节修正奇偶校验后解密并去 padding。

### 求解步骤

3DES 解密核心：

```python
from Crypto.Cipher import DES3
from Crypto.Util.Padding import unpad

def fix_des_parity(b):
    return b | 1 if bin(b).count("1") % 2 == 0 else b & 0xfe

ciphertext = bytes.fromhex("507CA9E68709CEFA20D50DCF90BB976C9090F6B07BA6A4E8")
key = bytes(fix_des_parity(b) for b in b"AFSAFCEDYCXCXACNDFKDCQXC"[:24])
second = unpad(DES3.new(key, DES3.MODE_ECB).decrypt(ciphertext), DES3.block_size)
print(second.decode())
```

得到后半段：

```text
0dcc509a6f75849b
```

结合脚本中前半段恢复结果，最终为：

```text
flag{13242238160dcc509a6f75849b}
```

## 方法总结

- 核心技巧：把二进制中的静态表转成 DWORD 运算约束，并识别第二段 3DES-ECB。
- 识别信号：校验逻辑按 DWORD 读取但只比较低字节时，可将未知量收缩到单字节枚举；看到 24 字节固定 key 和块密文时要考虑 3DES。
- 复用要点：DES/3DES key 可能需要奇偶校验修正；写脚本时应直接从附件二进制提取表，避免依赖旧的绝对路径。
