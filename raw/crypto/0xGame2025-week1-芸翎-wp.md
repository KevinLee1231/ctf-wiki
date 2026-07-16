# 芸翎

## 题目简述

题目包含两个连续阶段：先爆破 4 字符前缀，使其与服务端给出的后缀拼接后满足指定 SHA-256；随后解一个“模数本身是素数”的 BabyRSA。密文在传输时使用小端序，必须按题目方式转成整数。

## 解题过程

### PoW

很多 CTF-Crypto 交互题都会设置一个 PoW（proof of work），其广泛用于区块链操作、服务器请求过滤等场景中。

用于 Crypto 交互题的动机与后者类似，即要求连接者必须完成一定的计算量（通常是哈希爆破）才接受其请求。通过控制爆破的难度，对于正常的访问请求其耗时是可控乃至可忽略的，同时能有效防范分布式拒绝服务（DDoS）攻击。

本题的 PoW 代码逻辑比较简单，爆破目标是一个 4 位含大小写字母/数字的随机字符串前缀，需要令整个字符串的 sha256 哈希值等于目标值。

遍历所有可能的字符串即可（耗时应在数秒至数十秒数量级），同时通过搓爆破代码可以初步接触一下 Python 的库导入及部分库的使用（如 `hashlib`、`itertools`）。

候选空间为 $62^4$，逐个计算 `sha256((prefix + suffix).encode()).hexdigest()`，与目标摘要相等时提交前缀。使用 pwntools 可以把 PoW 与后续参数接收串成一次自动化交互。

### BabyRSA

通过 proof_of_work 后会将 flag 通过一个类似 RSA 的加密给出，区别在于此处的 N 是单个质数。

根据欧拉函数定义，$p$ 是质数时存在 $\varphi(p)=p-1$，可以直接计算获得私钥 $d$，随后解密步骤都与“常见”的 RSA 一致。

另外留意密文 `c` 的传输形式：十六进制先还原为字节，再按小端序转成整数参与模幂运算。这里的字节序是题目协议的一部分。

### 完整求解脚本

```python
from pwn import *
from itertools import product
from hashlib import sha256
import string


io = process(['python3', 'task.py'])

def bypass_proof():
    io.recvuntil(b'XXXX+')
    latter = io.recvuntil(b')',drop=True).strip().decode()
    io.recvuntil(b'==')
    hashval = io.recvline().strip().decode()
    for tp in product(string.ascii_letters+string.digits,repeat=4):
        pred = ''.join(tp)
        res = pred + latter
        if sha256(res.encode()).hexdigest() == hashval:
            io.sendlineafter(b'XXXX:',pred.encode())
            break


if __name__=='__main__':
    bypass_proof()

    io.recvuntil(b'=')
    n = int(io.recvline().strip().decode())
    io.recvuntil(b'=')
    e = int(io.recvline().strip().decode())
    io.recvuntil(b'=')
    c = bytes.fromhex(io.recvline().strip().decode())

    io.close()
    phi = n - 1
    d = pow(e,-1,phi)
    m0 = pow(int.from_bytes(c,'little'),d,n).to_bytes(253)
    print(m0)
```

脚本先完成 PoW，再读取 $n,e,c$。由于 $n$ 是素数，直接使用 $\varphi(n)=n-1$，计算 $d=e^{-1}\bmod(n-1)$ 并解密。若结果看似随机，应优先核对密文输入的 `little` 字节序，而不是怀疑 RSA 推导。

## 方法总结

- 核心技巧：小规模 SHA-256 PoW 穷举，以及素数模数下使用 $\varphi(p)=p-1$ 的 RSA 解密。
- 识别信号：固定长度未知前缀与已知哈希构成 PoW；RSA 输出中的模数通过素性判断后确认不是 $pq$。
- 复用要点：自动化交互要精确同步提示符；RSA 参数读取后先判断模数结构，并严格复现密文字节序。
