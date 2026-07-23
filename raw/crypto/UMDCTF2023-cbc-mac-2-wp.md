# CBC-MAC 2

## 题目简述

本题在上一题的 CBC-MAC 后追加一个 16 字节长度块：

```python
ct = AES.new(key, AES.MODE_CBC, iv=b"\x00" * 16).encrypt(
    msg + long_to_bytes(len(msg) // 16, 16)
)
tag = ct[-16:]
```

服务仍允许查询任意整块长度消息，并要求伪造一个未查询消息。追加长度能够阻止最直接的扩展，但长度块与普通消息块使用同一 CBC 链，攻击者可以通过多次查询拼出相同的内部状态。

## 解题过程

记 CBC 的状态更新为

$$
H_x(h)=E_k(h\oplus x),
$$

并以 $L_j$ 表示数值为 $j$ 的 16 字节长度块。先查询三条消息：

$$
\begin{aligned}
M_1&=0\Vert F\Vert L_2\Vert0\Vert F,\\
M_2&=0\Vert F,\\
M_3&=0\Vert0,
\end{aligned}
$$

其中 $0$ 是全零块，$F$ 是全 `ff` 块。记所得标签为 $t_1,t_2,t_3$。

由于服务会自动追加长度，$M_2$ 实际处理的分组为
$0\Vert F\Vert L_2$，所以处理完 $M_1$ 的前三个显式分组后，内部状态恰为 $t_2$。类似地，$M_3$ 的完整认证串为
$0\Vert0\Vert L_2$，标签为 $t_3$。

构造伪造消息

$$
M'=0\Vert0\Vert L_2\Vert(t_2\oplus t_3)\Vert F.
$$

它同样有 5 个显式分组，服务最终会追加 $L_5$。前三块处理后状态为 $t_3$，第四块使状态变成

$$
E_k\bigl(t_3\oplus(t_2\oplus t_3)\bigr)=E_k(t_2).
$$

这正好等于 $M_1$ 处理完第四块 `0` 后的状态。之后两条链都继续处理
$F\Vert L_5$，所以最终标签相同：

$$
\operatorname{MAC}(M')=\operatorname{MAC}(M_1)=t_1.
$$

对应构造为：

```python
m1 = zero + ff + length_2 + zero + ff
m2 = zero + ff
m3 = zero + zero

t1 = mac_query(m1)
t2 = mac_query(m2)
t3 = mac_query(m3)

forged = zero + zero + length_2 + xor(t2, t3) + ff
submit(forged, t1)
```

服务接受后返回：

```text
UMDCTF{W3lp_l00k5_l!k3_I_n33d_t0_ch4ng3_th1s_ag41n_s4d_f4ce_3m0j1_927323}
```

## 方法总结

- 核心技巧：利用 CBC 状态可控拼接，通过两个查询标签的异或把伪造链同步到另一条已知链的中间状态。
- 识别信号：长度虽被追加，但仍作为普通末尾分组进入同一个 CBC-MAC，且攻击者能查询多种长度时，不能直接认为长度扩展问题已经消失。
- 复用要点：证明伪造时应逐块写出链值，明确哪些标签可作为可控内部状态；只凭“CBC-MAC 可扩展”套用上一题公式会失败。
