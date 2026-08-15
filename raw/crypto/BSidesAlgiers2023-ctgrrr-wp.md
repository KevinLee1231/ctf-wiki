# CTGrrr

## 题目简述

附件提供已知明文 `file`、对应密文 `enc`，以及一份自制 AES-CTR 实现。密钥和初始计数器没有提供，但源码中的计数器更新逻辑存在严重缺陷：每处理一个 16 字节分组，程序会把计数器的每个字节都独立加 2，而不是对整个 128 位整数做带进位加法。

## 解题过程

CTR 模式对第 $i$ 个分组生成密钥流

$S_i=\operatorname{AES}_K(C_i)$

并计算 $E_i=P_i\oplus S_i$。源码中的更新函数如下：

```c
void increment_counter(unsigned char *counter, int inc)
{
    int i = BLOCK_SIZE - 1;
    int stop = 0;
    while (i >= 0 && !stop) {
        counter[i] = (counter[i] + inc) % 256;
        i--;
    }
}
```

`stop` 从未改变，所以 16 个字节都会执行模 256 加 2。对任意初值而言，一个字节经过 $128$ 次更新就会回到原值，整个计数器也随之重复。每个 AES 分组为 16 字节，因此密钥流周期是

$128\times16=2048\text{ bytes}$。

flag 被插入在文件后半部分，前 2048 字节仍与附件中的已知明文对齐。先计算这 2048 字节的密钥流，再按周期解密整个密文即可：

```python
from pathlib import Path

known = Path("file").read_bytes()
cipher = Path("enc").read_bytes()

period = 128 * 16
stream = bytes(cipher[i] ^ known[i] for i in range(period))
plain = bytes(byte ^ stream[i % period] for i, byte in enumerate(cipher))

start = plain.index(b"shellmates{")
end = plain.index(b"}", start) + 1
print(plain[start:end].decode())
```

输出为：

```text
shellmates{a3S_Ctr_l0sT_inCr3m3nt}
```

源码中的 `FLAG` 常量与附件密文中的实际 flag 不一致，而且仓库版本还把插入 flag 的调用注释掉了；因此应以发布附件 `file` 与 `enc` 的可复核解密结果为准。

## 方法总结

CTR 的安全性依赖计数器输入绝不重复。一旦计数器出现短周期，相同位置的密钥流就会反复使用，已知明文可以直接给出 $S=P\oplus E$，进而解密所有同余位置。本题的决定性缺陷不是 AES 本身，而是错误计数器让密钥流每 2048 字节重复。最终 flag 为 `shellmates{a3S_Ctr_l0sT_inCr3m3nt}`。
