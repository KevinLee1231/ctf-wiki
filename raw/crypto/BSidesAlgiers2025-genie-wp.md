# BSidesAlgiers2025 - Genie

## 题目简述

题目是 CSIDH 风格的签名服务，目标是通过验证接口直接触发后门 flag：

- 通过 `server.sage` 提供交互菜单：
  - `1`：先签名再验签
  - `2`：仅验签
- 只要提交 `message == "shellmates{Just_To_Make_Sure}"` 的有效签名，服务会输出 flag。
- 服务器定义了 `SCHEME(primes, S=16)`，每次密钥包括 16 个长度为 74 的向量；`choice=2` 时只公开 `Points_list`（点对）与参数，因此题目本质是“从公开点恢复密钥并可伪造签名”。

## 解题过程

### 1. 观察服务行为与可利用接口

`challenge/server.sage` 明确 `choice==2` 的执行路径：

```python
msg_to_verify = input("Enter a message to verify: ")
sig_input = ast.literal_eval(input("Enter the signature as a dictionary: "))
verify_message(msg_to_verify, sig_input)
```

`verify_message` 要求：
- `scheme.verify(msg,sig)` 为真；
- 同时 `msg == "shellmates{Just_To_Make_Sure}"` 才输出 flag。

### 2. 从公开点恢复 secret-key 条件（服务侧结构）

`scheme.keygen()` 输出每个子密钥对应一组公开点；把返回结构单独写成完整表达式如下：

```python
public_data = {
    "Points_list": [(P.xy(), PP.xy()) for _, P, PP in self.PK],
    "params": {"n": self.n, "S": self.S},
}
```

其中每组包含同一条椭圆曲线上的两点 `P, PP`。`solution/sol.sage` 中的 `get_curve` 直接由两个点反推该曲线参数：

$$
a = \frac{y_1^2-y_2^2-(x_1^3-x_2^3)}{x_1-x_2}
$$

再可用于判定两点是否在同一 $\ell_i$-子群上以及核元方向：

- `same_subgroup(P,PP,\ell)`：判断 $\left(\frac{p+1}{\ell}\right)P$ 与 $\left(\frac{p+1}{\ell}\right)PP$ 的 Weil 配对是否为 1；
- `kernel_sign(P,PP,p,\ell)` 用 Frobenius 判定符号并返回 $\pm 1$。

`get_secret_key` 对每个主子群 $\ell_i$ 聚合符号，得到

$$
\text{sk}[i][0] = \sum_{pairs} \ker\_sign_i(pair)
$$

并保留 74 维向量 `vec[i]`。按照 `scheme._gen_secret_key` 的递推关系，可以从该向量重建整组密钥：

```python
def rebuild_secret_keys(vec, S, sign):
    current = list(vec)
    secret_keys = [[0] * len(current), current.copy()]
    step = [-sign(value) for value in current]
    for _ in range(S - 2):
        current = [current[i] + step[i] for i in range(len(current))]
        secret_keys.append(current.copy())
    return secret_keys
```

所以只要 `SK[1]`，其余主钥匙都可直接构造。

### 3. 构造验证密钥并伪造签名

`solution/sol.sage` 不需要照搬服务端带噪声的正常签名函数，而是直接构造满足验证方程的干净响应。核心函数为：

```python
def forge_signature(secret_keys, msg, rounds):
    b_list, commit_curves = sample_ephemeral_randoms(rounds)
    c_list = derive_challenges(msg, commit_curves, rounds)
    responses = []
    for b, challenge in zip(b_list, c_list):
        key_vector = secret_keys[challenge]
        responses.append([
            b_i - key_i for b_i, key_i in zip(b, key_vector)
        ])
    return {"c": c_list, "r": responses}
```

验证器从公开曲线 `PK[c]` 出发再施加响应 $r=b-SK[c]$。由于 CSIDH 群作用按指数向量相加，结果正好回到临时承诺曲线 $E_b$，所以验证器重新派生出的挑战仍等于提交的 `c_list`。这与 `server.verify` 的重计算完全对应：

$$
c' = \mathrm{SHAKE}\big(E_j \| msg\big) \bmod (S-1)+1
$$

并对每个 `c_j` 重建 `recomputed_curve = csidh.group_action(base, r_j)`，最终比较 `c == c'`。

服务端正常签名路径中的随机噪声不是验证必需条件；验证器只检查重建曲线与 Fiat–Shamir 挑战是否一致。恢复 `SK[c]` 后，官方 solver 使用上述无噪声响应即可直接通过验证。

### 4. 可执行链条

```bash
# 以题目源码目录为工作目录
sage solution/sol.sage
```

脚本内置交互：
- 连接脚本中的占位服务地址；复现比赛实例时应替换为实际主机与端口；
- 发送 `2`（verify-only）
- 读取服务端返回 `Points_list`
- 发送目标消息 `shellmates{Just_To_Make_Sure}`
- 发送构造的 `sig`
- 输出 flag。

仓库内 `challenge/flag.txt` 给出的最终结果是：

`shellmates{Weil_Pairing_and_Frobenius_u_get_torsion_group_Kernel_Sign}`

说明：脚本使用 Sage 与 pwntools 的远程连接，代数恢复流程已与官方 solver 对齐；本轮没有重新启动完整服务做端到端交互。公开的 [Genie 解题记录](https://medium.com/@saad_alqarni/bsides-algiers-2025-genie-crypto-write-up-ddeff4f9f9dd) 也采用“投影到小素数阶扭点子群、判断线性相关性与核符号、重建密钥”的同一主线，正文已经包含其必要机制。

## 方法总结

- 核心技巧：用公开点构造 `ℓ_i` 子群信息，恢复 `SK[1]` 后重建整个签名密钥结构，再伪造可验证签名。
- 识别信号：
  - `verify-only` 的验证服务在给定 challenge 下常可做伪造；
  - `Points_list` 暴露了足够的代数特征；
  - 若验证中 `sign` 与 `verify` 共享同构流程，能复刻构造。
- 复用要点：本题属于“数学结构可逆 + 服务接口可验签”的典型 CSIDH/椭圆-签名伪造模式，后续可直接复用于含 `Point_list` 与 `H` 派生挑战的一类题目。
