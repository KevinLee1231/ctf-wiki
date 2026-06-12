# SU_Lock

## 题目简述
题目是隐藏锁屏程序，题干要求在 Windows 10/11 测试模式虚拟机环境中分析。样本是多层 Windows 恶意/锁屏逆向：Inno Setup 外壳、Rust overlay 解包、锁屏程序和内核驱动。真正的 flag 校验在驱动层，外层安装器密码和中间载荷都只是进入下一层的门。

## 解题过程
外层样本是一个伪装成 Everything_Setup_1.4.1.exe 的 Inno Setup 安装器，真正逻辑一共
分三层：

1. Inno Setup 外层安装器：脚本里藏了第一层密码。

2. 第二层 Rust 程序：解析自身 overlay，解出伪装载荷。

3. 第三层锁屏程序 + 内核驱动：真正的 flag 校验在驱动里。

外层：Inno Setup 安装器

strings 很容易看出它是 Inno Setup：
- Inno Setup Setup Data (6.7.0)
- Inno Setup Messages (6.5.0) (u)
- ccPascal
- ccStdCall

资源里还能看到 PACKAGEINFO ，说明这是 Delphi/Inno 的标准壳。

题目弹了密码页，但这题最关键的点是：密码不是靠爆破，而是脚本自己填进去的。

我这里的做法是把 Inno 的 setup0 数据块和其中的 _CompiledCode 抽出来，再用 IFPS（Inno
Pascal Script）解析。脚本里能恢复出这些函数名：
- !MAIN
- ISTESTMODEENABLED
- ISAVRUNNING
- SHOULDDEPLOYMALWARE
- CurPageChanged

其中 CurPageChanged 最关键。它会在密码页出现时：

1. 设置 WizardForm.PasswordEdit.Text

2. 再直接调用 WizardForm.NextButton.OnClick

也就是“自动帮你填密码并点下一步”。密码就是：

suctf

脚本里还能看到两个很有意思的检查：

- ISTESTMODEENABLED
- ISAVRUNNING

ISAVRUNNING 里会通过 WMI 查询一些进程名，例如：

- 360tray.exe
- 360sd.exe
- ProcessHacker.exe
- wireshark.exe

这也解释了为什么题目描述里会特地强调：

- 在虚拟机里做

- 打开 Windows Test Mode

因为后面要落一个未签名驱动，不开 Test Mode 很难正常跑通。

通过 Inno 数据流解包后，能拿到两份主要文件：
- file0.bin ：官方 Everything.exe

位 Rust 程序（后文叫 stage2 ）• file1.bin ：一个自定义的 64

其中 file0.bin 基本就是烟雾弹，真正要分析的是 file1.bin 。

file1.bin 是个 64 位 PE，字符串里能看到：
- sample.pdb
- src\main.rs
- zip-0.6.6\src\aes.rs
- zip-0.6.6\src\read.rs
- bzip2
- flate2
- sha1
- zstd
- CreateServiceA
- OpenServiceA
- StartServiceA
- DeleteService

这说明几件事：

1. 它是 Rust 写的。

2. 它会处理一个带 AES 的压缩包。

3. 它有加载/控制服务的能力，明显在为驱动做准备。

stage2 的 PE 末尾还带了一段 overlay。把 PE 本体截掉后，overlay 开头长这样：

43 58 03 04 ...

也就是：

CX\x03\x04

不是正常 ZIP 的 PK\x03\x04 。

继续扫完整个 overlay，可以发现：

- 两个 CX\x03\x04 （两个 local header）

- 两个 CX\x01\x02 （两个 central directory entry）

- 一个 CX\x05\x06 （一个 EOCD）

所以它本质上就是一个 ZIP，只不过把头部签名做了替换。

overlay 里第一个文件名能直接看到是：

1.wct

第二个同理会看到类似 2.jzi 。

这两个扩展名做 ROT13 后分别变成：
- wct -> jpg
- jzi -> wmv

也就是说，程序把压缩包里的文件伪装成图片/视频。

在 stage2 的 .rdata 能找到字符串：

SUCTF2026

第二层还原载荷时用到的关键字。实际分析下来，它承担的是：

- ZIP/AES 解包口令

- 以及后续隐藏载荷恢复时使用的关键字

也就是第二层的“钥匙”。

恢复 overlay 里的内容，做法很直接：

1. 取出 stage2 的 overlay。

2. 把所有 ZIP 头签名从 CX 改回 PK 。

3. 文件名做一次 ROT13，还原成正常扩展名。

4. 用 SUCTF2026 解包/还原。

解出来会得到两份伪装文件，继续还原后得到：

- 用户态锁屏程序

- 内核驱动

题目提示“lock-screen program”非常准确：

- 用户态程序负责 UI/输入

- 驱动负责校验

这也是为什么单看用户态程序时，你会发现它没有把 flag 明文写死，而是依赖设备通信。

驱动创建设备后，用户态通过下面这个设备名通信：

\\.\CtfMalDevice

最关键的两个控制码是：

- 0x222004
- 0x222008

IOCTL 0x222004

这个 IOCTL 会返回后续算法用到的常量：
- delta = 0x9e376a8e
- key[0] = 0xdeadbeef
- key[1] = 0xcafebabe
- key[2] = 0x1337c0de
- key[3] = 0x0badf00d

IOCTL 0x222008

这个 IOCTL 会把输入按一个 XXTEA-like 的 32 位分组算法处理，然后和驱动内置的 10 个 dword 密文
常量做比较。

换句话说：

- UI 程序只负责把你输入的字符串丢给驱动

- 驱动负责真正加密并比较

所以这题的正解思路不是“硬跑锁屏”，而是：

1. 逆向驱动算法

2. 把内置密文反推出明文

驱动里比较的是 10 个 DWORD ，所以明文也是按 DWORD 组织的一串字符。把驱动里的那套加密过
程抄出来，再写一个逆过程，就能把最终字符串恢复出来。

算法形态非常像 XXTEA / Block TEA 的变体：

- 使用 delta = 0x9e376a8e

- 每轮会混合相邻 dword

- 会索引 4 个 key 常量

因此流程就是：

1. 从 0x222004 拿到 delta 和 key[4]

2. 从 0x222008 校验逻辑抄出 10 个密文 dword

3. 写逆过程把 10 个 dword 还原成字节串

4. 按 ASCII 拼回 flag

### Exp:

```python
import struct

MASK = 0xffffffff
DELTA = 0x9e376a8e
KEY = [0xdeadbeef, 0xcafebabe, 0x1337c0de, 0x0badf00d]

CIPHER = [
0xDBDDACB6,
0xED7199EE,
0x6E403589,
0xED74E4C7,
0x05AD8C30,
0xFF8AA14A,
0x033D9788,
0xFDCAAD29,
0x8E0FCA1B,
0x61463F4F,
]

def u32(x):
return x & MASK

def mx(z, y, sum_, p, e, k):
return u32(
(((z >> 5) ^ (y << 2)) + ((y >> 3) ^ (z << 4)))
^ ((sum_ ^ y) + (k[(p & 3) ^ e] ^ z))
)

def btea_decrypt(v, k):
n = len(v)
rounds = 6 + 52 // n
sum_ = u32(rounds * DELTA)

# 关键：y 需要在循环里持续传递
y = v[0]

while sum_ != 0:
e = (sum_ >> 2) & 3

for p in range(n - 1, 0, -1):
z = v[p - 1]
y = v[p] = u32(v[p] - mx(z, y, sum_, p, e, k))

z = v[n - 1]
y = v[0] = u32(v[0] - mx(z, y, sum_, 0, e, k))

sum_ = u32(sum_ - DELTA)

return v

def dwords_to_bytes_le(v):
return b"".join(struct.pack("<I", x) for x in v)

def main():
plain_dw = btea_decrypt(CIPHER[:], KEY)
plain = dwords_to_bytes_le(plain_dw)

print("[+] dec dwords:", [f"0x{x:08x}" for x in plain_dw])
print("[+] raw :", plain)
print("[+] flag :", plain.decode("ascii"))

if __name__ == "__main__":
main()
```

```
python -u "exp.py"
[+] dec dwords: ['0x54435553', '0x4a537b46', '0x32414d43', '0x58412d33',
'0x33514d38', '0x382d5549', '0x53434855', '0x2d30394f', '0x314d4351',
'0x7d4c3053']
[+] raw : b'SUCTF{SJCMA23-AX8MQ3IU-8UHCSO90-QCM1S0L}'
[+] flag : SUCTF{SJCMA23-AX8MQ3IU-8UHCSO90-QCM1S0L}
```

## 方法总结
- 核心技巧：多层安装器/驱动逆向
- 识别信号：样本伪装安装器，资源和 overlay 中继续嵌套可执行文件或驱动。
- 复用要点：逐层提取密码和载荷，定位最终驱动 IOCTL/加密校验，再反解输入。
