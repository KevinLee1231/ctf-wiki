# SUCTF2026-Lock

## 题目简述

题目是多层 Windows 锁屏样本：Inno Setup 安装器释放 Rust stage2，stage2 再恢复并加载用户态锁屏程序与内核驱动，真正的 flag 校验由驱动 VM 完成。题面要求在开启 Test Mode 的 Windows 10/11 虚拟机中分析，是因为样本会注册未签名驱动并执行进程注入；若无需动态取证，应优先静态提取，避免在宿主机直接运行附件。

题目有随机化附件。本文对应 `Everything_Setup2.zip`，最终 flag 长度为 40 字节；其它附件的驱动 VM 密文和 flag 会不同，不能机械复制本文常量。

## 解题过程

### 1. 从 Inno Setup 外壳取得 stage2

字符串中的 `Inno Setup Setup Data`、`Inno Setup Messages`、`ccPascal` 与 `PACKAGEINFO` 可以确认外层格式。[Inno Unpack GUI 2.2.4](https://github.com/jrathlev/InnoUnpacker-Windows-GUI/releases/tag/iu_2_2_4) 是官方题解使用的 Inno Setup 查看/提取工具；该发布版内置 `innounp.exe 2.67.4`，可列出安装器资源，但题目安装包的直接提取仍受密码保护。

官方预期路线是运行安装器后用取证工具提取 `Locksetup.exe`。总 PDF 还给出了一条纯静态补充路线：解析 `setup0` 中的 `_CompiledCode`，在 Inno Pascal Script 里找到 `CurPageChanged`。密码页出现时，它会把 `WizardForm.PasswordEdit.Text` 设为 `suctf`，再主动触发 `NextButton.OnClick`，所以密码不是爆破目标。

同一脚本还检查：

- `ISTESTMODEENABLED`：确认 Windows Test Mode；
- `ISAVRUNNING`：通过 WMI 查找 `360tray.exe`、`360sd.exe`、`ProcessHacker.exe`、`wireshark.exe` 等进程；
- `SHOULDDEPLOYMALWARE`：决定是否继续释放后续载荷。

解包后主要得到官方 `Everything.exe` 和 Rust 编写的 `Locksetup.exe`，前者是伪装，后者才是 stage2。

### 2. 正确还原 Rust stage2 的 overlay 和内嵌载荷

Rust 程序的 PE overlay 以 `43 58 03 04`，即 `CX\x03\x04` 开头。不要把它理解成若干独立的手工补丁：源码实际从最后一个该魔数处截取整个 overlay，然后对全部字节执行 ROT13。

这一操作会同时完成：

- ZIP local header：`CX\x03\x04` → `PK\x03\x04`；
- central directory 与 EOCD：`CX\x01\x02`、`CX\x05\x06` → 对应 `PK` 签名；
- 文件名：`1.wct` → `1.jpg`，`2.jzi` → `2.wmv`。

恢复后的 ZIP 由 Rust `zip` crate 直接读取并释放到 `%APPDATA%`，不需要再用 `SUCTF2026` 作为 ZIP/AES 口令。`SUCTF2026` 的真实用途是 RC4 密钥：stage2 用它解密编译进程序的 `ENCRYPTED_DRIVER` 和 `ENCRYPTED_LOCKER`，随后注册驱动服务，并把锁屏程序注入 `svchost.exe`。总 PDF 将这两层用途混在了一起，按源码应以上述数据流为准。

锁屏程序负责全屏 UI、键盘钩子和输入，驱动负责进程保护、文件过滤及最终校验。两者通过设备 `\\.\CtfMalDevice` 通信。

### 3. 从驱动 VM 提取密钥和目标密文

用户态源码给出两个控制码：

```text
CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)
  = 0x222004  -> IOCTL_GET_SECRETS

CTL_CODE(FILE_DEVICE_UNKNOWN, 0x802, METHOD_BUFFERED, FILE_ANY_ACCESS)
  = 0x222008  -> IOCTL_VERIFY_FLAG
```

`0x222004` 让驱动 VM 向输出缓冲区写入算法常量，同时记录需要保护的进程 PID；`0x222008` 则让另一段 VM bytecode 逐 DWORD 读取用户态传来的密文并与立即数比较。

VM 只有五类操作：

```text
0x10  MOV Rn, imm32
0x20  STORE_OUT [Raddr], Rvalue
0x30  LOAD_IN Rdst, [Raddr]
0x40  ASSERT_EQ Rleft, Rright
0xff  HALT
```

解释 `g_VmCode_GetSecrets` 可得：

```text
delta = 0x9e376a8e
key   = [0xdeadbeef, 0xcafebabe, 0x1337c0de, 0x0badf00d]
```

对 `Everything_Setup2` 的 `g_VmCode_Verify` 取所有写入 `R2`、随后用于 `ASSERT_EQ` 的立即数，可得十个目标 DWORD：

```text
8da1e7b1 caa432e5 6eec27bc efc12b53 fa7505c2
54ac88a6 2f96ad99 77741a15 3e8673c1 c2b9f282
```

附件是随机化的，因此应从当前驱动的 bytecode 提取这些数，而不是使用总 PDF 中另一条逆向路径的 `DBDDACB6 ...` 密文。

### 4. 逆官方 XXTEA 变体

锁屏程序只接受长度恰为 40 的 UTF-8/ASCII 输入，将其原地解释成十个小端 `uint32_t`，再执行 Block TEA/XXTEA 变体：

```c
#define MOD_MX \
    (((z >> 4 ^ y << 3) + (y >> 2 ^ z << 5)) \
    ^ ((sum ^ y) + (key[(p & 3) ^ e] ^ z)))
```

这里的移位组合与标准 XXTEA、以及总 PDF 中参赛队为另一组中间密文写出的公式均不同，必须以用户态源码为准。逆运算如下：

```python
import struct

MASK = 0xFFFFFFFF
DELTA = 0x9E376A8E
KEY = [0xDEADBEEF, 0xCAFEBABE, 0x1337C0DE, 0x0BADF00D]
CIPHER = [
    0x8DA1E7B1, 0xCAA432E5, 0x6EEC27BC, 0xEFC12B53, 0xFA7505C2,
    0x54AC88A6, 0x2F96AD99, 0x77741A15, 0x3E8673C1, 0xC2B9F282,
]


def decrypt_variant(words, key, delta):
    count = len(words)
    rounds = 6 + 52 // count
    total = rounds * delta & MASK
    y = words[0]

    for _ in range(rounds):
        e = total >> 2 & 3
        for p in range(count - 1, 0, -1):
            z = words[p - 1]
            part1 = ((z >> 4) ^ (y << 3)) & MASK
            part2 = ((y >> 2) ^ (z << 5)) & MASK
            part3 = ((total ^ y) + (key[(p & 3) ^ e] ^ z)) & MASK
            y = words[p] = (words[p] - ((part1 + part2) ^ part3)) & MASK

        z = words[-1]
        part1 = ((z >> 4) ^ (y << 3)) & MASK
        part2 = ((y >> 2) ^ (z << 5)) & MASK
        part3 = ((total ^ y) + (key[e] ^ z)) & MASK
        y = words[0] = (words[0] - ((part1 + part2) ^ part3)) & MASK
        total = (total - delta) & MASK

    return words


plain_words = decrypt_variant(CIPHER[:], KEY, DELTA)
plain = struct.pack("<10I", *plain_words)
print(plain.decode("ascii"))
```

输出为：

```text
SUCTF{SJCMA23-AX8MQ3IU-8UHCSO90-QCM1S0L}
```

## 方法总结

- 多层安装器先按“安装脚本 → stage2 → 用户态载荷 → 驱动”建立数据流，不要在第一层密码页或官方诱饵程序上停留。
- 对自定义归档格式应先寻找统一变换；本题对整个 overlay 做 ROT13，一次同时恢复 ZIP 签名与文件名。
- 外层的 `SUCTF2026` 是 RC4 密钥，不是 ZIP 口令；只有顺着源码调用点追踪，才能避免把相邻层的常量用途混淆。
- IOCTL 只是传输边界。真正的秘密和目标密文藏在两段极小 VM bytecode 中，先解释操作码比盲目动态调试驱动更直接。
- 随机化附件必须从当前样本提取 VM 立即数；XXTEA 变体也必须逐项核对移位、加法/XOR 优先级和字节序。
