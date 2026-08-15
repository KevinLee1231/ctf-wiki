# hodor-hodor-hodor

## 题目简述

题目公布了一段十六进制密文、已知 flag 形态 `shellmates{hOdOr_?}`，以及一份故意残缺的生成代码。公开代码只展示：随机 12 字节密钥会 XOR flag 的前 11 字节和最后一个字节，而真正插入中间密文的 `secret_function` 被留空、对应调用也被注释。官方 solver 则给出了部署版本中间区域的逆变换，因此可利用已知前后缀跳过不可恢复的一次性 XOR 部分。

## 解题过程

公开的 `xor` 逻辑只产生 12 字节：

```python
def xor(flag, key):
    output = bytearray()
    for i in range(11):
        output.append(flag[i] ^ key[i])
    output.append(flag[-1] ^ key[11])
    return output
```

完整密文却远长于 12 字节，且源码留下了如下被注释的插入操作：

```python
# encrypted[11:11] = secret_function(flag, SECRET_KEY)
```

这说明密文结构应为：随机 XOR 的 11 字节前缀、中间的 `secret_function` 输出、随机 XOR 的末字节。随机密钥没有泄露，前后两部分不能直接解密；但它们对应的明文已经由 flag 格式确定为 `shellmates{` 和 `}`。

官方 solver 表明，中间区域第 $i$ 个密文字节满足

$$
c_i\equiv m_i+i\pmod{256},
$$

因此逐字节执行

$$
m_i\equiv c_i-i\pmod{256}
$$

即可恢复 flag 主体：

```python
enc = bytes.fromhex(
    "cbcd6ab3321d77b7c1efa5735b715d816f7a80725a49798b774d66925096427e6d86639198846e57946d897f94728d736061a492968a8a968f81899ab0856eb487a996a298ad7abba77dae8fb5c1977fbeb0b587a2b4989cb7aeada0c0bca4ced2c095b1a7d7bfd7dcb29ab91b"
)

flag = bytearray(b"shellmates{")
for i in range(11, len(enc) - 1):
    flag.append((enc[i] - i) % 256)
flag.append(ord("}"))

print(flag.decode())
```

使用仓库自带脚本实际运行得到：

```text
shellmates{hOdOr_in_F4ct_4Lw4y$_MeAnt_H0lD_ThE_D00r_bUT_WHO_tH0uGhT_Th4t_4dDitI0n_c4N_BE_USEd_For_3NCrYptI0N}
```

需要注意：仓库里的 challenge 文件不是能够重新生成给定密文的完整部署源码，`secret_function` 的正向实现确实缺失。上述变换关系来自同仓库官方 solver，并由其对给定密文的可打印输出验证；不能反过来声称公开 challenge 已完整实现该函数。

## 方法总结

- 核心技巧：识别密文分段，只逆转中间区域的按位置加法，利用已知 flag 格式补回随机 XOR 覆盖的前后缀。
- 识别信号：生成代码输出长度与给定密文长度不一致、存在被注释的切片插入、官方逆向脚本跳过首尾字节时，应先还原数据布局。
- 复用要点：位置型加法必须使用密文中的绝对索引 $i$，并在模 256 下运算；不要把缺失的正向函数凭空补成“已知源码”。
