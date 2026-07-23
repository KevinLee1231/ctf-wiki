# Triple DES

## 题目简述

题目并非标准 3DES，而是用三个独立 DES-CBC 密钥连续加密三次。每层都会随机生成一个 8 字节 IV，并把 IV 前置到该层密文：

```python
data = pad(flag, 8)
for key in keys:
    cipher = DES.new(key, DES.MODE_CBC)
    data = cipher.iv + cipher.encrypt(data)
```

服务还允许提交任意至少 32 字节、长度为 8 的倍数的密文。它按相反密钥顺序逐层取出前 8 字节作为 IV 并解密，最后只返回 PKCS#7 去填充是否成功。这仍然是 padding oracle；三层 CBC 只增加了数据布局，没有消除 CBC 的可塑性。

## 解题过程

### 把三层结构压缩成单块 oracle

向服务提交四个 8 字节块 $Y_0\|Y_1\|Y_2\|Y_3$。第一层逆向解密得到三个块，第二层得到两个块，第三层最终只得到一个 8 字节明文块。

只修改最外层 IV $Y_0$ 时，差分会依次落到下一层的 IV，最终原样异或到最后的明文块。若
$Y'_0=Y_0\oplus\Delta$，则最终有

$$
P'=P\oplus\Delta.
$$

因此，截取最终密文中任意连续 32 字节，都能构成针对相应原始明文块的独立 oracle 查询；需要控制的仍只是这 32 字节中的第一块。

### 逐字节猜测可打印明文

flag 长度为 40 字节，正好是五个 DES 分组。官方脚本以 8 字节为步长，从最终密文截取五个 32 字节窗口，刻意不处理最后那个纯填充分组：

```python
windows = [
    encrypted_flag[i:i + 32]
    for i in range(0, len(encrypted_flag) - 32, 8)
]
```

对每个窗口从末字节向前恢复。若当前要制造值为 $r$ 的合法填充，并猜测原明文字节为 $g$，就在可控的第一块对应位置异或
$g\oplus r$。当猜测正确时，该位置会变为 $r$，已恢复后缀也被同步改成
$r$，oracle 返回 `yes`：

```python
for recovered in range(8):
    pad_value = recovered + 1

    for guess in string.printable.encode():
        wanted[7 - recovered:] = bytes([pad_value]) * (recovered + 1)
        known[7 - recovered] = guess

        crafted = xor(xor(window[:24], wanted), known) + window[24:]
        if oracle(crafted):
            break
```

这里 `wanted` 和 `known` 虽然分配为 24 字节，但脚本只修改前 8 字节；后 16 字节保持原值，用于维持后两层解密链。

五个窗口依次恢复出：

```text
UMDCTF{padding_oracle_with_extra_steps?}
```

## 方法总结

- 核心技巧：分析多层 CBC 的差分传播，确认最外层 IV 的异或改动会穿过三次解密，最终直接作用于单个原始明文块。
- 识别信号：无论嵌套多少层，只要解密端暴露最终填充是否合法，并且某个前置块可控，就要检查是否仍保留 CBC 可塑性。
- 复用要点：先用最小合法密文长度把复杂封装化成“一次查询只检查一个明文块”；若明文字母表未知，应枚举完整 256 字节，而不能依赖本题 flag 全为可打印字符这一条件。
