# SUCTF2026-MirrorBus9

## 题目简述

MirrorBus-9 是一个黑盒半双工工业总线模拟器。客户端只能发送协议命令并读取延迟帧，无法直接观察 9 维隐藏状态。目标是利用可重复的黑盒实验，把状态控制到 `ARM` 门限；成功后服务返回 `CHAL`，再提交三元组 `PROVE p1 p2 p3` 获取 flag。

题目涉及的核心命令为 `RESET`、`ENQ`、`ARM`、`COMMIT`、`POLL` 和 `PROVE`。连接 banner 中的 `gift` 是明确设计的假 flag，不能提交。

## 解题过程

### 1. 确认 half-duplex 协议节奏

先执行最小交互：

```text
HELP
STATUS
ENQ INJ 0 1
COMMIT
POLL 16
```

由回包可确定：

- `ENQ` 返回 `QOK` 只表示命令入队，不表示已经执行；
- `COMMIT` 才推进内部 tick 并执行队列；
- 观测帧不会随命令立即回显，必须通过 `POLL` 拉取；
- 正确节奏是 `ENQ -> COMMIT -> POLL`，不能把延迟帧错配给最近一条命令。

公共部署对每条 TCP 连接派生独立 seed，但同一连接里的 `RESET` 会把隐藏状态、队列、tick、gate 和确定性噪声全部恢复到相同初态。因此所有采样、求解和最终证明都应在同一条连接内完成。

### 2. 把 `ARM_FAIL` 视为二维仿射残差

源码中的隐藏状态属于有限域：

```text
x_t ∈ F_65521^9
```

`ARM` 实际检查两条线性约束：

```text
<v1, x_t> = target1  (mod 65521)
<v2, x_t> = target2  (mod 65521)
```

失败时不会直接返回原始距离，而是先定义：

```text
d1 = target1 - <v1, x_t>
d2 = target2 - <v2, x_t>
```

再通过一个可逆的 2×2 矩阵混合为 `ARM_FAIL` 帧中的 `sig` 和 `aux`。矩阵本身未知并不影响求解：在固定 transcript 下，`(sig, aux)` 对注入值仍是有限域上的仿射函数。

协议还支持 `ROT`、`MIX`、`BIAS`，但标准解只需最直接的 `INJ lane value`。对候选 lane 对 `(i, j)` 固定执行：

```text
RESET
ENQ INJ i a
ENQ INJ j b
ENQ ARM
COMMIT
POLL 64
```

### 3. 用三次实验恢复控制方向

对同一对 lane 分别采样：

```text
d0 = ARM_FAIL(0, 0)
di = ARM_FAIL(1, 0)
dj = ARM_FAIL(0, 1)
```

定义两条控制方向：

```text
e_i = d0 - di
e_j = d0 - dj
```

如果：

```text
det = e_i.x * e_j.y - e_i.y * e_j.x  mod 65521
```

不为 0，则这两条方向线性无关，可以解出注入量 `(vi, vj)`：

```text
[e_i.x  e_j.x] [vi]   [d0.x]
[e_i.y  e_j.y] [vj] = [d0.y]  (mod 65521)
```

枚举 `0 <= i < j < 9`，找到首个行列式非零的 lane 对即可。用模逆写成显式公式：

```text
vi = (d0.x * e_j.y - d0.y * e_j.x) / det
vj = (e_i.x * d0.y - e_i.y * d0.x) / det
```

所有加减乘除都在模 `65521` 下进行。

### 4. 触发 `CHAL`

把求出的 `(vi, vj)` 代回同一实验模板。若直接返回 `CHAL`，说明两维残差都已归零；若仍返回新的 `ARM_FAIL = r`，则解：

```text
[e_i e_j] · [ci, cj]^T = r
```

并更新：

```text
vi <- vi + ci
vj <- vj + cj
```

重复少量几次即可吸收模型中的固定偏移。官方参考实现最多修正 4 次。

成功帧形如：

```text
F ... sig=<sig> aux=<aux> tag=CHAL nonce=<nonce> ttl=<ttl>
```

这里的 `sig`、`aux` 已经是最终证明所需投影，不应再拿当前 ARM 状态另做线性猜测。

### 5. 计算 `PROVE`

证明参数定义为：

```text
p1 = CHAL.sig
p2 = CHAL.aux
p3 = CRC16-CCITT(f"{nonce}:{p1}:{p2}")
```

CRC 使用初值 `0xffff`、多项式 `0x1021`，输入是上述 ASCII 字符串，不做最终异或：

```python
def crc16_ccitt(data: bytes) -> int:
    crc = 0xffff
    for byte in data:
        crc ^= byte << 8
        for _ in range(8):
            crc = ((crc << 1) ^ 0x1021) & 0xffff if crc & 0x8000 else (crc << 1) & 0xffff
    return crc
```

参赛队记录中的一组实际 `CHAL` 为：

```text
nonce = 175a6f7bf012
p1    = 60056
p2    = 41938
```

对 `b"175a6f7bf012:60056:41938"` 计算得到 `p3 = 3459`，提交：

```text
PROVE 60056 41938 3459
```

服务返回：

```text
OK cmd=PROVE status=PASS flag=SUCTF{mb9_file_only_flag_runtime_hardened}
```

### 6. 参赛队 PDF 中的非预期路线

参赛队没有识别出 CRC 输入格式，而是利用部署参数 `MB9_PROVE_ATTEMPTS=7` 和 `RESET` 的确定性：同一连接中重建相同 `CHAL`，固定 `p1=CHAL.sig`、`p2=CHAL.aux`，每轮枚举 7 个 `p3`，失败后再次 `RESET`。这样可把 16 位空间分批穷举，最终同样命中 `3459`。

这条路线在当时部署上有效，但它比官方解慢，且依赖尝试次数、命令预算和 reset 行为；重新连接还会改变 session seed。正文应以“二维差分控制 + 明确的 CRC16-CCITT”作为标准解，暴力仅作为协议重放风险的补充观察。

## 方法总结

本题的关键是把“无回显、延迟、噪声”拆开验证，而不是看到复杂状态机就猜内部公式。`RESET` 提供了受控实验条件，`ARM_FAIL` 提供二维观测，三次基点采样便能恢复每对控制量的仿射方向，再用一个 2×2 模线性方程把残差归零。

可复用的黑盒分析顺序是：先确认协议节奏，再验证同 transcript 可重放；选最简单的控制原语做基线和单位扰动；检查控制方向是否线性无关；最后只依据成功挑战帧定义计算证明。不要混淆 `QOK`、`OBS`、`ARM_FAIL` 和 `CHAL`，也不要把 banner 中过早出现的字符串当作真实 flag。
