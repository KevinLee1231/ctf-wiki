# GlacierCTF 2024 qamu

## 题目简述

qamu 给出一个 64 位 ELF、`vm_enc.bin` 和仅供本地验证的假 flag。主程序模拟“机密计算”环境：它把加密程序拆成 4 KiB 页面，按内置页序列逐页执行；执行加密页触发 `SIGSEGV` 后，异常处理器临时解密当前页、赋予执行权限，运行其中一条有效指令，再把页面重新加密。

最终目标是恢复这段被分页保护的 license checker，并构造 32 字节的“license key + MAC”，令虚拟程序返回非零。仓库官方 WP 只有占位内容，下面的过程由官方 exploit、题目二进制以及逐页解密资料交叉还原。

## 解题过程

### 1. 分清宿主与客体内存

`main()` 读取 `vm_enc.bin` 后固定映射五段内存：

```text
0x30000000  临时页
0x40000000  宿主寄存器状态
0x50000000  客体寄存器状态及最终返回值
0x60000000  下一页入口指针
0x70000000  重新排列后的加密 VM 页面
```

`.data` 中 `0x4040a0` 起始的 574 个 qword 是页面编号序列。加载循环对每个编号 `p` 复制：

```c
memcpy((void *)((p + 0x70000) << 12),
       vm_enc + p * 0x1000,
       0x1000);
```

程序从标准输入读取 `0x20` 字节，再把每个输入字节扩展为一个 qword 保存。VM 返回后只检查 `*(uint64_t *)0x50000000 != 0`；成立就读取并打印 `flag.txt`。

### 2. 还原按需解密机制

对仍处于密文状态的页执行会产生 `SIGSEGV`。信号处理器取故障地址所在的页，构造 32 字节 RC4 key：

```text
little_endian(page_address, 8)
25 fa 5b c3 52 a7 88 d5 5c 15 ff 16 e0 ac 36 2a
e0 d1 37 40 59 83 7d 55
```

然后原地 RC4 解密 4096 字节，并用 `mprotect(..., PROT_READ|PROT_WRITE|PROT_EXEC)` 允许执行。每个明文页的开头只有一条属于客体程序的有效 x86-64 指令，紧接着 `jmp r15` 返回宿主；后续无效字节会促使未经处理的页面继续触发异常。

宿主在每条客体指令前后分别保存两套寄存器，随后用同一 RC4 流重新加密当前页并改回不可执行。保护依赖的不是未知密码，而是二进制里完整公开的页地址与常量，所以可以离线复现。

### 3. 离线拼接 license checker

对 `vm_enc.bin` 的每一页用相应地址生成 key，解密后反汇编第一条指令，再按 `0x4040a0` 的页面序列拼接。核心伪代码如下：

```python
from Crypto.Cipher import ARC4
from struct import pack

SUFFIX = bytes.fromhex(
    "25fa5bc352a788d55c15ff16e0ac362a"
    "e0d1374059837d55"
)

def decrypt_page(blob, page_no):
    page = blob[page_no * 0x1000:(page_no + 1) * 0x1000]
    address = (0x70000 + page_no) << 12
    return ARC4.new(pack("<Q", address) + SUFFIX).decrypt(page)
```

拼接后得到约 2077 字节、574 条指令的检查程序。输入指针位于 `rdi`；固定格式约束包括：

```text
key[3]  == 'P'
key[5]  == '-'
key[11] == '-'
key[17] == '-'
key[23] == ':'
```

其余约束数量较多，适合把拼接出的代码映射到独立地址，用 angr 将 32 个输入 qword 设为符号量，并约束终态 `rax != 0`。不同模型可能给出不同但同样有效的输入；一份公开分析得到：

```text
AHMPQ-30009-AABJW-qkjhh:\x60\xbcK\xc5am\xc0\x16
```

官方 exploit 使用的是另一组满足约束的 32 字节输入：

```python
payload = bytearray(b"ANMPQ-10206-ADBHQ-qij3h:AAAAAAAA")
payload[24:32] = bytes.fromhex("bb 0a 2f 07 f4 0b c6 58")
```

向服务发送该 payload 后，VM 返回非零，宿主进入 `valid` 分支并打印比赛 flag。

仓库随题目的 `flag.txt` 明确是：

```text
gctf{FAKE_FAKE_FAKE_FAKE_FAKE_FAKE}
```

它只用于本地测试，不是比赛答案。官方仓库、官方 exploit 及现存的完整解题资料均未记录远端真实 flag，因此这里不把假 flag 冒充最终答案。逐页解密、寄存器切换和符号执行的补充分析见 [Sudeep Singh 的 qamu 题解](https://www.sudeepvision.com/blog/glacier_ctf_2024_qamu_reverse_engineering_challenge/)；其决定性信息已完整归纳在正文中，无需依赖外链才能复现。

## 方法总结

qamu 的障碍是“页级 RC4 自修改执行 + 一指令一页”的表示方式，而不是不可获得的密钥。先从异常处理器恢复页密钥，再按内置页序列抽取首条指令，就能把复杂的运行时保护降维成普通 license checker，最后交给符号执行求解。审校时必须区分本地假 flag 与远端 flag：有效 payload 已有官方证据，但真实比赛 flag 未被公开材料保存。
