# z3kapig

## 题目简述

服务实现了两方 CGGMP21 门限 ECDSA。玩家扮演 Party 0，服务端扮演 Party 1，依次完成：

```text
密钥生成
  -> Paillier 辅助参数生成
  -> 预签名
  -> 签名与验签
```

服务端私钥份额为 $x_1$。只有至少完成一次成功签名后，玩家才可以调用 `guess_key` 猜测 $x_1$；猜中后服务返回 flag。

攻击点不在 ECDSA 签名公式本身，而在恶意参与方可提交的 Paillier 模数及其零知识证明。通过构造含有许多已知小素因子的模数，并为每个小素数重复一次恶意预签名，可以恢复

$x_1\bmod p_i$

最后用 CRT 重建服务端完整私钥份额。

## 解题过程

### 1. 先明确服务端状态机

`main.py` 严格限制正常协议消息的阶段和顺序，但 `guess_key` 是特殊操作：

```python
if action == "guess_key":
    return self.handle_guess_key(data)
```

猜测前还要满足：

```python
if not self.signing_completed:
    return {
        "correct": False,
        "message": "You must complete at least one successful signing round before guessing.",
    }
```

所以不能只泄漏同余后中途退出。每一轮恶意预签名都要构造出能被服务端接受的后续证明，并至少完成一次成功验签。服务完成签名阶段后会回到 phase 3，允许在同一组长期密钥和辅助参数下继续预签名，这正好可以对多个小素数重复攻击。

远程连接开始还有 26 位工作量证明。官方脚本读取 JSON 中的 challenge 和 difficulty，求出满足条件的 nonce 后再进入协议；整个协议默认有 600 秒超时。

### 2. 提交含有许多小素因子的 Paillier 模数

正常 Paillier 模数应由两个满足要求的大素数相乘。官方脚本却预先构造：

$N=p_{\mathrm{large},0}\,p_{\mathrm{large},1}\prod_i p_i$

其中用于泄漏的 $p_i$ 是约 18 位的小素数，例如 `262147`、`262151`、`262153` 等。脚本知道 $N$ 的完整分解，因此仍可计算：

$\varphi(N)=\prod_{p\mid N}(p-1)$

以及 Paillier 解密所需的 $\lambda(N)$。

辅助参数阶段的 `ProofMod.verify()` 会检查：

- $N$ 为奇数；
- $W$ 具有所需的二次剩余性质；
- 80 组 $N$ 次根和四次根关系成立；
- 各元素位于合法区间。

但它没有单独证明“$N$ 恰好是两个满足协议要求的安全素数之积”。已知多因子分解后，攻击者可以用 $\varphi(N)$ 计算所需根，逐项构造能通过验证的 `ProofMod`。官方 `fake_mod_proof()` 已内置一组模数、因子和证明构造，并在本地执行：

```python
assert mod.verify(ssid, N)
```

随后照常生成 `ProofPrm` 与 `ProofFac`，使恶意 $N$ 被服务端接受为 Party 0 的 Paillier 公钥。

### 3. 针对一个小素数构造特殊密文

固定 $N$ 的一个小素因子 $p$，令：

$M=N/p$

正常预签名第一轮应发送 Party 0 临时份额 $k_0$ 的 Paillier 密文。攻击脚本把本地状态设为：

$k_0=0$

但实际发送：

$K_0=\operatorname{Enc}_N(M)$

还需要让 `ProofEnc` 看起来像是在证明“$K_0$ 加密了 0”。Paillier 使用生成元 $g=1+N$，且：

$g^N\equiv1\pmod{N^2}$

若 Fiat-Shamir 挑战 $e$ 满足：

$e\equiv0\pmod p$

则 $eM=eN/p$ 是 $N$ 的倍数，于是验证方程中的明文差异被消去：

$g^{eM}\equiv1\pmod{N^2}$

官方 `fake_enc_proof()` 不断增加承诺中的 `alpha` 并重新计算哈希挑战，直到：

```python
if e % prime == 0:
    z1 = e * k + alpha
    z2 = pow(rho, e, N) * r % N
    z3 = e * mu + gamma
    return ProofEnc(S, A, C_, z1, z2, z3)
```

这里不是逆向哈希，而是按约 $p$ 次试验做 Fiat-Shamir grinding。由于 $p$ 只有约 18 位，成本可以接受。

### 4. MtA 输出只取决于 $x_1\bmod p$

服务端收到 $K_0=\operatorname{Enc}(M)$ 后，在预签名 `round2()` 中分别与自己的随机量 $\gamma_1$、长期私钥份额 $x_1$ 执行 MtA：

```text
D   对应 gamma_1
_D  对应 x_1
```

以 `_D` 为例，Party 0 已知恶意 Paillier 私钥，可解密得到：

$d\equiv x_1M+\beta\pmod N$

其中 $\beta$ 是服务端为了隐藏乘积而加入的大随机掩码。

因为 $M=N/p$：

$(x_1+p)M\equiv x_1M\pmod N$

所以密文只携带 $x_1\bmod p$。令：

$r=x_1\bmod p$

则：

$d\equiv rM+\beta\pmod N$

如果能约束 $\beta$，就能从这个近似子集和关系恢复小范围的 $r$。

### 5. 从 Aff-g 证明响应泄漏掩码高位

服务端随 MtA 输出发送 `ProofAffg`。其响应字段之一为：

$Z_2=e\beta+b$

其中 $e$ 是约曲线阶大小的 Fiat-Shamir 挑战，$\beta$ 是接近 $N$ 位宽的 MtA 掩码，而证明盲因子满足 $0<b<N$。

因此：

$\left\lfloor Z_2/e\right\rfloor=\beta+\left\lfloor b/e\right\rfloor$

右侧误差至多约为 $N/e$，只影响 $\beta$ 的低位。官方脚本取：

```python
shift = N.bit_length() - 250
observed_high = (proof.Z2 // e) >> shift
```

得到掩码约 250 个高位，并枚举一个很小的进位修正。写成：

$\beta=\beta_{\mathrm{high}}2^s+\beta_{\mathrm{low}}$

其中：

$0\le\beta_{\mathrm{low}}<2^s$

再结合 Paillier 解密关系，可得精确整数方程：

$\beta_{\mathrm{high}}2^s-d+rM+\beta_{\mathrm{low}}+kN=0$

未知量满足：

$0\le r<p,\qquad0\le\beta_{\mathrm{low}}<2^s$

### 6. 用不等式格同时恢复余数和低位掩码

官方脚本把常数项、$M$、单位低位和 $N$ 组成格：

```python
const = known_high * (1 << shift) - dec
mat = Matrix(ZZ, [
    [const, 1, 0, 0],
    [M,     0, 1, 0],
    [1,     0, 0, 1],
    [N,     0, 0, 0],
])

lb = [0, 1, 0, 0]
ub = [0, 1, prime, 1 << shift]
ans = solve_inequality(mat, lb, ub)
```

目标向量第一坐标为 0，第二坐标固定为 1；第三、第四坐标分别限制为 $r$ 与 $\beta_{\mathrm{low}}$ 的合法范围。`lll_cvp.py` 先规约格，再求满足区间约束的近向量。

取回候选后仍进行代数验证：

```python
mask = (dec - residue * M - high_part) % N
if 0 <= residue < prime and 0 <= mask < mask_bound:
    return residue, mask
```

对 `_D` 执行该过程得到 $x_1\bmod p$；对 `D` 也执行一次，主要是恢复 $\gamma_1$ 对应的完整掩码，供后续伪造一致的预签名份额使用。

### 7. 修补后续证明，完整走完一次协议

泄漏同余后，Party 0 的本地状态仍声称 $k_0=0$，而服务端刚才实际对 $\operatorname{Enc}(N/p)$ 做了 MtA。若直接使用库生成的后续消息，双方的 `delta`、`chi` 和曲线点检查会不一致。

官方脚本使用恢复出的两个掩码重算本地份额：

```python
alpha_gamma = gamma_mask % q
alpha_xi = xi_mask % q

delta_0 = (
    gamma_0 * k_0
    + alpha_gamma
    + beta_gamma
) % q

chi_0 = (
    x_0 * k_0
    + alpha_xi
    + beta_xi
) % q
```

并将 $v\delta_0$ 设为与 $k_0=0$ 一致的无穷远点/零倍点。

后续 `ProofLogstar` 仍面对同一个问题：真实密文是 $\operatorname{Enc}(N/p)$，声明标量却是 0。解法与 `ProofEnc` 相同，再次调整承诺，直到 Fiat-Shamir 挑战满足：

$e\equiv0\pmod p$

这样就能构造通过验证的预签名 round-out 消息。签名阶段关于 $k_0$ 的 Logstar 证明也同样处理。

脚本使用固定消息：

```python
msg = b"Hello, world!"
```

双方签名并互相验签。服务端返回 `verify: true` 后，`signing_completed` 被置为真，协议状态回到下一次预签名入口。

### 8. 对多个素数重复并用 CRT 重建 $x_1$

对恶意模数中的每个小素数重复上述流程，记录：

$r_i=x_1\bmod p_i$

然后计算：

```python
crt_modulus = math.prod(primes)
x1 = int(crt(primes, residues)[0] % crt_modulus)
```

所有 $p_i$ 互素，所以中国剩余定理给出：

$x_1\equiv x_{\mathrm{CRT}}\pmod{\prod_i p_i}$

最后在曲线阶 $q$ 范围内枚举：

$x_{\mathrm{CRT}}+k\prod_i p_i<q$

逐个调用：

```json
{
  "action": "guess_key",
  "data": {
    "guess": 123456
  }
}
```

服务端允许错误猜测后继续会话；命中真实 $x_1$ 时响应中出现 `correct: true` 和 flag。

远程运行时，协议模块位于 `attachment`，格辅助代码位于 `solve`。可从附件目录启动，使两边都在 Python 搜索路径中：

```bash
cd attachment
RHOST='challenge-host' PORT='1337' sage ../solve/solve.py
```

脚本依赖 SageMath、pwntools、SymPy，以及附件中的 `crypto`、`ecdsa` 和 PoW 模块。

## 方法总结

完整攻击链为：

1. 正常完成门限 ECDSA 密钥生成；
2. 在辅助参数阶段提交一个已知完整分解、含多个小素因子的 Paillier 模数，并构造能通过的 ProofMod；
3. 对每个 $p_i$ 发送 $\operatorname{Enc}(N/p_i)$，通过挑选 $e\equiv0\pmod{p_i}$ 的 Fiat-Shamir transcript，把它伪装成 $\operatorname{Enc}(0)$；
4. 解密服务端 MtA 输出，得到只依赖 $x_1\bmod p_i$ 的关系；
5. 从 Aff-g 响应 $Z_2=e\beta+b$ 提取掩码高位，再用 LLL/CVP 恢复余数和掩码低位；
6. 用恢复出的掩码修补 `delta`、`chi`，并以同样的挑战整除技巧伪造后续 Logstar 证明；
7. 完成有效签名，重复收集同余；
8. CRT 重建服务端长期私钥份额并调用 `guess_key`。

本题说明，门限协议不能只验证“每一条群等式看起来成立”。参与方公钥的结构、证明语句的范围，以及 Fiat-Shamir 挑战与恶意小因子的组合都属于安全边界。若模数结构证明不足，原本只应泄漏加法份额的 MtA 可以退化为按小素数查询长期私钥的同余 oracle。
