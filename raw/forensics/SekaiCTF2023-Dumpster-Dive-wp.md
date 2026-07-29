# Dumpster Dive

## 题目简述

题目给出攻击者 Linux 主机的内存转储，并说明该主机与另一题 Infected 的网络流量有关。目标不是直接在内存中搜索 flag，而是恢复攻击者曾在 Python 交互环境里使用的密钥材料，再用它解密 Infected 流量中的响应。

内核版本为 Linux 5.x，Volatility 2 无法正确解析；需要为对应内核准备符号表并使用 Volatility 3。整条证据链如下：

```text
内存镜像
  -> Bash 历史定位 Python 交互会话
  -> 进程列表确定 PID 2055
  -> 转储 Python 进程堆
  -> 恢复变量 c、k、type
  -> c XOR k 得到 RSA 私钥
  -> 解密 Infected 的响应数据
```

## 解题过程

### 定位目标进程

先加载与镜像内核匹配的 Volatility 3 Linux 符号。使用 `linux.bash` 检查 shell 历史时，可以发现攻击者启动过 `python3` 交互解释器；再用 `linux.pslist` 列出进程，目标 Python 进程的 PID 为 `2055`。

不同 Volatility 3 版本的插件全名略有差异，官方解法使用的关键操作可以概括为：

```text
linux.bash
linux.pslist
linux.proc --pid 2055 --dump
```

这里不能只对整个镜像运行一次 `strings` 后搜索 `SEKAI{`。题目把决定性材料留在 Python 堆对象中，先缩小到 PID 2055 能显著减少噪声，也能保留变量之间的关联。

### 从 Python 堆中恢复变量

检查转储出的堆区，重点寻找 Python 字符串对象及交互变量名。可以观察到三个相关变量：

```text
c
k
type
```

`c` 是被掩码后的长字节串，`k` 是等长 XOR 掩码，`type` 则帮助确认该对象应按私钥材料解释。对两个十六进制串逐字节异或：

```python
def xor_equal(left: bytes, right: bytes) -> bytes:
    assert len(left) == len(right)
    return bytes(a ^ b for a, b in zip(left, right))

private_key_pem = xor_equal(
    bytes.fromhex(c_hex),
    bytes.fromhex(k_hex),
)
```

结果以标准 PEM 标记开头：

```text
-----BEGIN RSA PRIVATE KEY-----
```

这同时验证了变量提取、字节顺序和 XOR 关系均正确。

### 解密关联流量

Infected 的 WebShell 会用只保存在攻击者侧的公钥加密命令输出，因此仅靠服务器文件和入站请求无法恢复响应。Dumpster Dive 恰好补齐了对应私钥。加载恢复出的 PEM 私钥，对抓包中 WebShell 返回的 Base64/RSA 密文逐块执行 PKCS#1 v1.5 解密：

```python
import base64
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import padding

private_key = serialization.load_pem_private_key(
    private_key_pem,
    password=None,
)

plaintext = b"".join(
    private_key.decrypt(base64.b64decode(chunk), padding.PKCS1v15())
    for chunk in response_chunks
    if chunk
)
print(plaintext.decode())
```

解密后得到：

```text
SEKAI{h0pe_y0u_enj0y3d_s0m3_l1nux_v0l}
```

## 方法总结

- 核心技巧：先用 shell 历史和进程列表建立行为线索，再定向转储 Python 堆，从运行时对象中恢复密钥。
- 识别信号：题面明确关联另一份 PCAP，而服务器端缺失响应私钥；这表明内存镜像中的目标是跨题补齐密钥，而不是独立搜索 flag。
- 复用要点：Linux 内存取证必须匹配内核符号和工具版本；恢复两个等长高熵变量后，可用预期文件头或 PEM 标记验证 XOR 结果。跨证据源题目还应明确每把密钥能解哪一方向的流量。
