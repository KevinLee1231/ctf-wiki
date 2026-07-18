# revme

## 题目简述

题目是一个 GHC 编译的 Windows 可执行文件。候选字符串先经过 Haskell parser combinator 解析，再依次接受九段独立哈希、48 字节置换与镜像异或、七组超递增背包和一个位置相关校验。赛事仓库标明本题没有单题 WP，总 PDF 也未收录；以下解法完全由公开的 `revme.hs` 源码推导，并用原校验逻辑逐项复核。

## 解题过程

### 解析格式与分段哈希

解析器只接受如下文法：

```text
RCTF{segment_..._segment}
```

内部必须恰好有 9 段；每段非空，字符只允许小写字母、数字或字面量 `?`。后续置换表长度为 48，所以完整输入也必须恰好 48 字节。扣除 `RCTF{}` 和 8 个下划线，九段字符长度之和为 34。

第 `i` 段使用独立的 `salt` 和 `target`。哈希初值与单步更新为：

```text
acc0 = salt * 1337
acc' = rol64(acc, 5) xor (ord(ch) + salt)
acc' = acc' + popcount(ord(ch)) * (salt + 3)  mod 2^64
```

这个变换对“给定末状态和末字符”可逆：

```text
prev = ror64(
    (cur - popcount(ch) * (salt + 3)) xor (ord(ch) + salt),
    5
)
```

因此对每个短段做双向枚举即可：正向枚举前半段并按中间状态建表，反向枚举后半段并查表。九段的长度依次为：

```text
7, 2, 5, 5, 3, 2, 3, 5, 2
```

分段哈希本身故意存在碰撞，例如第一段的 `h4sk3ll`、`h4zw3ll`、`he?k3ll` 都命中同一目标；第四段至少有 `dup3r` 与 `dupjd`，第八段至少有 `r1ght` 与 `t4njp`。所以这里不能见到一句通顺文本便提前停止，还要继续使用全局校验消歧。

### 反演七组背包

`finalCheck` 先按固定的 48 项置换表重排完整输入，再生成字节流密钥：

```text
cur0 = 0xac
cur(i+1) = (cur(i) * 73 + 41) mod 256
key = cur1, cur2, ...
```

随后逐字节执行：

```text
xored[i]  = permuted[i] xor key[i]
added[i]  = xored[i] + 0x3d  mod 256
mirror[i] = added[i] xor added[47-i]
```

48 个 `mirror` 字节转为 MSB-first 位流，末尾再追加一个由九段长度折叠得到的 guard 字节。392 位被分成 7 组，每组 56 位，并以同一组 56 个权重求和后与七个目标比较。

权重序列满足超递增性质，所以每个目标从最大权重向最小权重贪心相减即可唯一恢复 56 位，不需要通用背包算法。七个目标还原出的 49 字节为：

```text
a4b01d0e6ce7577c744d27a00ee7413dec7339fb8a470df6
f60d478afb3973ec3d41e70ea0274d747c57e76c0e1db0a4
d6
```

前 48 字节正好前后对称，对应 `mirror`；最后的 `d6` 是 guard。按源码的折叠式验证长度：

```text
acc = 0
for length in [7,2,5,5,3,2,3,5,2]:
    acc = (acc * 7 + length) mod 256
# acc == 0xd6
```

将已知格式字符、每段哈希候选以及 `mirror[i]` 的成对关系代回置换，就能排除上述碰撞，得到唯一能通过七组背包的自然语言候选：

```text
RCTF{h4sk3ll_1s_sup3r_dup3r_fun_t0_r3v_r1ght_??}
```

末尾两个问号是 flag 的字面字符，不是待替换的通配符。最后再计算：

```text
checksum = xor(ord(ch) * (index + 1) for index, ch in enumerate(input))
```

结果为 `1209`，与源码目标一致；九段哈希、guard、七个背包和总校验也全部通过，因此最终 flag 即为：

```text
RCTF{h4sk3ll_1s_sup3r_dup3r_fun_t0_r3v_r1ght_??}
```

若要从公开 Haskell 源码重新编译，需要补上其中遗漏的 `import Data.List (foldl')`；这不影响已发布二进制及上述算法还原，但会影响源码直接编译。

## 方法总结

本题不需要从 GHC 运行时海量样板中逐函数硬啃。定位到 parser、常量表和纯函数校验后，可以按可逆性分层处理：分段滚动哈希用中间相遇，超递增背包用贪心，镜像异或用置换后的成对约束，最后以位置校验消歧。尤其要防止两个直觉错误：哈希命中不代表分段唯一，`??` 也不是尚未求出的字符。
