# oracle

## 题目简述

题目给出原生 `oracle` 二进制和 `feed.bin`。数据先经过两层自定义 AES 包装，再进入 MWC 异或的归档；其中的验证类又由自定义 VM 执行。目标是恢复 32 字节 ticket，使 VM 的 8 个寄存器经过 8 轮可逆变换后等于目标常量，并用该 ticket 解密 flag。

## 解题过程

### 解开 feed 容器

二进制 `.rodata` 中保存了 32 个经过异或掩码的 16 字节 AES key。还原 key table 后，每一层按 64 字节头部选择三个索引：

```text
i1 = header[(block + 11) & 63] % 32
i2 = header[(block + 17) & 63] % 32
i3 = header[block & 63] % 32
derived_key = AES-ECB-DEC(key[i3], key[i1])
plaintext   = AES-CBC-DEC(derived_key, iv=key[i2], block)
```

连续解两层后得到 `ARC\x01` 归档。用头部种子初始化两个 multiply-with-carry 状态，对归档正文逐字节异或，即可解析出名称、长度、内容组成的 TLV 条目。

### 逆向验证 VM

条目 `b` 使用 `BC3B` 类魔数，正文还需以魔数派生种子再做一次 MWC 解密。其常量池前三项分别为：

- 16 个 32 位 round key；
- 8 个目标寄存器；
- `umdctf-v2026-unseal-salt`。

VM 将 ticket 按小端拆成 8 个 `uint32`，执行 8 轮异或、加法、循环左移、寄存器混合和乘法。所有操作均可逆：加法改为减法、`ROL` 改为 `ROR`，异或原样撤销，奇数乘数 `0x5abc7f01` 则乘其模 $2^{32}$ 的逆元。按第 7 轮到第 0 轮、每轮指令逆序处理目标寄存器，即可唯一恢复 ticket。

### 解密 flag

验证 `forward(ticket, RK) == TARGET` 后计算：

```python
flag_km = hash_x(ticket + salt)
key = flag_km[:16]
iv = flag_km[16:32]
```

对归档条目 `g` 做 AES-CBC 解密并移除 PKCS#7 填充，得到：

```text
UMDCTF{oh_no_my_prediction_market_feed_has_been_compromised_what_ever_will_i_do}
```

若要让原程序接受 ticket，还需封装为 `TKT\x01 || uint32_le(32) || ticket`，Base64 后放入 `BEGIN/END MARKET TICKET` 文本。

## 方法总结

层数多不等于每层都难。先按数据边界逐层拆开 AES、MWC、TLV 和 VM，再利用验证轮函数全部可逆这一事实从目标状态倒推输入。每层魔数、长度、正向寄存器结果和 AES 填充都能作为局部验收点，避免错误一直传播到最后。
