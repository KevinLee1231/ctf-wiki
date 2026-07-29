# X64 Extension

## 题目简述

附件包含一个静态链接、去符号的 x86-64 ELF 和 `flag.txt.enc`。程序读取 `flag.txt`，使用汇编实现的加密函数生成 160 字节密文。算法主体使用 AES-NI，看起来像 AES-256-CBC，但密钥扩展常量被修改，直接调用标准 AES 库无法解密。

题目名中的 “X64 Extension” 指的就是 `aesenc`、`aesenclast`、`aeskeygenassist` 等 x86 AES 指令扩展。

## 解题过程

### 1. 定位加密函数与固定参数

在二进制中搜索 AES-NI 指令，可以定位到连续 13 次 `aesenc` 和一次 `aesenclast` 的块加密函数。调用者还以立即数形式在栈上构造 IV 和 256 位 key：

```text
IV  = ff fe fd fc fb fa f9 f8 f7 f6 f5 f4 f3 f2 f1 f0
KEY = 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f
      10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
```

每块进入 AES 轮函数前，状态先与上一密文块和第一轮密钥异或；第一块使用固定 IV。加密结束后把当前密文保存为下一块 IV，因此工作模式是 CBC。程序还按块尾缺少的字节数填充相同数值，即 PKCS#7 padding。

### 2. 找出非标准密钥扩展

标准 AES-256 需要 15 个 128 位轮密钥。该二进制仍使用正常的 AES-256 扩展结构，但七次 `aeskeygenassist` 的轮常量不是标准的：

```asm
aeskeygenassist xmm3, xmm2, 0x13
aeskeygenassist xmm3, xmm2, 0x33
aeskeygenassist xmm3, xmm2, 0x37
aeskeygenassist xmm3, xmm2, 0xba
aeskeygenassist xmm3, xmm2, 0xda
aeskeygenassist xmm3, xmm2, 0x55
aeskeygenassist xmm3, xmm2, 0x66
```

标准 AES-256 对应位置通常使用 `01, 02, 04, 08, 10, 20, 40`。因此不能只把 key 和 IV 填进 PyCryptodome；必须复刻原扩展，把 Rcon 序列替换为：

```text
13 33 37 ba da 55 66
```

除这七个立即数外，字移位、S-box、前半密钥递推以及对后半 128 位使用 `aeskeygenassist ..., 0x00` 的逻辑都与常规 AES-256 相同。

### 3. 构造逆向轮密钥并执行 CBC 解密

先按修改后的 key schedule 生成加密轮密钥 $K_0,\ldots,K_{14}$。AES-NI 的 `aesdec` 形式要求对中间轮密钥做逆 MixColumns：

```text
D0  = K0
Di  = AESIMC(Ki), 1 <= i <= 13
D14 = K14
```

对每个 16 字节密文块 $C_i$ 执行：

```text
state = C_i XOR K14
state = AESDEC(state, AESIMC(K13))
...
state = AESDEC(state, AESIMC(K1))
state = AESDECLAST(state, K0)
P_i   = state XOR (IV if i == 0 else C_{i-1})
```

官方汇编解密器的关键循环正是：

```asm
pxor       xmm0, round_key_14
aesdec     xmm0, inv_round_key_13
; ... 省略其余中间轮
aesdec     xmm0, inv_round_key_1
aesdeclast xmm0, round_key_0
pxor       xmm0, previous_cipher_or_iv
```

最后读取明文末字节作为 padding 长度，确认末尾每个字节都相同并删除。附件密文为 160 字节，共 10 个块。

若直接使用官方解密实现，可将 `AES.asm` 与一个读取 `flag.txt.enc` 的 C++ 包装器编译：

```bash
nasm -f elf64 -o AES.o AES.asm
g++ -c main.cpp -o main.o
g++ -o solve AES.o main.o
./solve
```

输出明文为：

```text
Hey Sekai CTF Player, I hope you are fine and are enjoying the CTF. Keep going, here is your reward! The flag is SEKAI{Pl34Se_It'5_jUs7_@_wAaaarmUp}
```

因此 flag 是：

```text
SEKAI{Pl34Se_It'5_jUs7_@_wAaaarmUp}
```

## 方法总结

识别出 AES-NI 指令并不等于可以直接使用标准 AES 解密。这里的轮函数、CBC 链和 PKCS#7 都是标准形式，唯一但决定性的变化是 AES-256 key schedule 的七个 Rcon 立即数。对比 `aeskeygenassist` 序列即可快速发现差异。

复现修改后的密钥扩展，再按 AES-NI 解密要求对中间轮密钥执行 `aesimc`，就能逐块解开密文。分析这类“魔改标准密码”题时，应分别核对工作模式、轮函数、轮数、key schedule 和 padding，不要只凭出现 `aesenc` 就套用库函数。
