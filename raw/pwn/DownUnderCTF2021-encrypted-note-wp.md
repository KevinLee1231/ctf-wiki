# DownUnderCTF 2021 - encrypted note

## 题目简述

服务用 64 位线性同余生成器（LCG）的输出逐块异或加密 88 字节栈上笔记。表面上是密码题，但最终障碍是内存破坏：程序对密文使用 `strlen` 和 `strchr` 查找 NUL，再从该位置追加并重新加密八字节，攻击者可借助可预测密钥流把追加位置逐步移动到笔记之外，泄露并覆盖栈 canary、保存的 `rbp` 和 PIE 返回地址。

## 解题过程

### 恢复 LCG

生成器满足：

$$
X_{n+1}=A X_n+B\pmod {2^{64}}.
$$

每次写入已知的八字节 `AAAAAAAA`，再读取其密文，异或即可得到连续状态 $X_1,X_2,X_3$。令 $m=2^{64}$，则：

$$
A=(X_3-X_2)(X_2-X_1)^{-1}\pmod m,
$$

$$
B=X_2-AX_1\pmod m.
$$

初始化把 $A$ 和 $B$ 都强制为奇数，使 $X_2-X_1$ 为奇数，可在模 $2^{64}$ 下求逆。核心代码为：

```python
from Crypto.Util.number import inverse

m = 2**64
A = (X3 - X2) * inverse(X2 - X1, m) % m
B = (X2 - A * X1) % m

def next_state():
    global X
    X = (A * X + B) % m
    return X.to_bytes(8, "little")

def preimage(wanted_ciphertext):
    assert len(wanted_ciphertext) % 8 == 0
    out = b""
    for offset in range(0, len(wanted_ciphertext), 8):
        block = wanted_ciphertext[offset:offset + 8]
        out += bytes(a ^ b for a, b in zip(block, next_state()))
    return out
```

`preimage` 计算“提交什么明文，才能在加密后得到指定密文”。提交数据本身不能过早出现 `\0`，否则 `encrypt` 的 `strlen` 会少处理区块；官方 solver 会向前推进 LCG，选择不会在关键位置产生零字节的状态。

### 把追加操作扩展到栈外

追加逻辑的问题集中在三行：

```c
char *dest = strchr(note, '\0');
strncpy(dest, app, 8);
dest[7] = '\0';
encrypt(dest);
```

先构造密文，使笔记的第一个 NUL 位于偏移 81。下一次追加覆盖 `81..88`，而偏移 88 已是笔记外、也是 canary 的首字节。重新加密会把原本为零的 canary 首字节变成非零，于是 `read_note` 的 `strlen` 越过笔记并输出 canary 后七字节；补回已知的首字节 `\0` 即得到完整 canary。

随后把密文中的第一个 NUL 依次向后移动，通过多次八字节追加跨过 canary 和保存的 `rbp`。`read_note` 最终泄露偏移 104 处的返回地址，减去二进制内返回点偏移 `0x169f` 即得到 PIE 基址。

### 覆盖返回链

得到 canary 与 PIE 后，目标栈布局为：

```text
offset 0x58: stack canary
offset 0x60: saved rbp
offset 0x68: ret gadget
offset 0x70: win
```

其中 `win` 直接执行 `system("/bin/sh")`。由于每次追加都会再消费一个 LCG 状态，写入前必须同步真实状态与本地预测状态；官方 solver 搜索未来状态，使关键块的末字节加密后为 NUL，从而既完成当前写入，又为下一次 `strchr` 留下准确落点。按顺序写入 `win`、恢复 canary 与栈填充、写入对齐用 `ret`，最后再恢复 canary 的零首字节。

选择菜单项 `0` 让 `vuln` 返回，canary 校验通过后执行 `ret; win`，得到 shell 与：

```text
DUCTF{c4n_1_g3t_a_p1zza_th4t_1s_h4lf_p3pper0n1_4nd_h4lf_ch33s3_ple4se!}
```

## 方法总结

可预测 LCG 本身只提供密钥流预测，真正把它升级为利用链的是“对二进制密文使用 C 字符串函数”。恢复 LCG 后，应把每个可控 NUL 当成游标：`strchr` 决定下一次写入位置，追加和再次加密则让游标跨越边界。混合 Crypto/Pwn 题应分别维护生成器状态、密文布局和栈布局，任何一次额外加密都会让后续预测整体错位。
