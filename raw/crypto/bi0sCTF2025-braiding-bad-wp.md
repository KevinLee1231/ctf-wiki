# Braiding Bad

## 题目简述

`chall.sage` 在 `BraidGroup(n)` 上构造：

$$
q = a \, p \, a^{-1}
$$

其中：

- `p` 是一个固定的“中心片段”；
- `q` 是用随机 `a` 共轭得到；
- `c1 = br * p * br^{-1}`；
- 内部再计算 `br * q * br^{-1}` 的右正规形哈希，并把 64 字节填充明文与该 SHA-512 摘要异或；最终输出把这段异或结果转成整数，字段名也叫 `c2`。

最终公开量是 `p`、`q`、`c1` 的 Tietze 词列表以及整数 `c2`。目标是从前三个编织群元素恢复哈希材料，再解开整数密文。

## 解题过程

官方 `admin/exp.sage` 的思路是反向重建 `a` 的等价分解：

1. 取 `u = p` 的 `Tietze` 列表，构造 `factors = [i-1]`；
2. 按索引分三类：
   - `< mid` 的生成元乘到 `x1`；
   - `> = mid` 的乘到 `x2`；
   - `mid` 区间统计中间计数 `cnt`；
3. `z = gs[mid-1]^cnt`，验证 `x1 * x2 * z == p`；
4. 用 `v (q)` 和 `w (c1)` 反解：

$$
r1=v\cdot z^{-1}\cdot x2^{-1},\quad r2=x1^{-1}\cdot w\cdot z^{-1},\quad r=r1\cdot r2\cdot z
$$

5. `h = SHA512(prod(r.right_normal_form()))`；
6. 对 `c2` 解码后逐字异或 `h`。

```sage
factors = [i - 1 for i in u.Tietze()]
...
z = gs[n // 2 - 1] ** cnt
r1 = v * z^-1 * x2^-1
r2 = x1^-1 * w * z^-1
r = r1 * r2 * z
```

```sage
h = hashlib.sha512(str(prod(r.right_normal_form())).encode()).digest()
message = "".join(chr(ord(dd[i]) ^ h[i]) for i in range(len(h)))
```

脚本恢复的是带随机字母前后缀的 64 字节明文；从中定位标准 flag 片段即可得到：

```text
bi0sCTF{8r41d_c0nju6473_m4573ry_f5b3f7c4}
```

## 方法总结

- 该题不是“解密困难性”，而是利用共轭群在 `Tietze` 表达下的结构性关系将 `br` 的核心参数提取出来。
- 验证链路建议按 `factors -> z -> x1/x2 -> r1/r2 -> h` 逐步对齐，不要从 `c2` 直接逆向，否则容易丢失 `right_normal_form` 与 `x^-1` 方向。
- 代码内的 `exp.sage` 已给出可复用参数推导脚本，按步骤执行即可直接到 `message`。
- 本次未重新运行 Sage 编织群计算；公式、官方脚本输出目标与 README 中的 flag 已做静态交叉核对。
