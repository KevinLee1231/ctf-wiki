# Baby Isogeny

## 题目简述

`chall.sage` 构造的是 SIKE/同源群环境：

- 固定两个基子群幂阶：$2^{216}$ 与 $3^{137}$；
- 服务器公开 `PA, QA, PB, QB` 与多组点/曲线不变量；
- 生成两段 flag 加密：`flag1` 与 `flag2` 分别对应 2-幂和 3-幂密钥分支；
- 服务端查询接口允许提交 `(public key + IV + CT)` 进行 300 次 `Good!/Bad!` 校验。

## 解题过程

源码给了完整攻防模型，官方攻击脚本 `admin/exploit/exp.sage` 拆为两步恢复：

1. **先取第一段密钥相关参数**（2-阶）
   `s.sage` 利用 `A_P_A / A_Q_A / E_A` 直接解出 2-幂秘密量：

   ```sage
   a = phiA_Q_A
   b = E_A(0)-x*phiA_PA
   y = discrete_log(b, a, ord=2^216, operation="+")
   ```

   由此构造 `flag1` 所需的 $j$ 不变量并解密第一段密文（`IV1/CT1`）。该段只是 oracle 所需的已知明文，解出的是诱饵 `bi0s{D3c0y_Fl4g}`，不是最终提交值。

2. **再恢复 3-阶参数（GPST）并解码第二段**
   `admin/exploit/exp.sage` 利用服务器提供的公开参数调用 `oracle(EA, φAPB, φAQB, EAB, flag1)`：

   - 先读全量公参（`P/Q`、各对 pair 不变量）；
   - 用 `attack(eA,eB,...)` 的 trit 迭代恢复 Alice/Bob 3-幂指数；
   - 攻击核心通过 `oracle` 多次提交构造出的 `(phiP,phiQ)`，配合 `getAmounts` 逐步恢复每一位三进制；
   - 收敛后补最后 3 个 trit 暴力枚举，得到 `alpha_`；
   - 用 `alpha_` 解 `flag2`，得到最终 flag `bi0s{B4by1s0g3ny_J1nv4r14nt_Pwn}`。

`oracle` 中的共享密钥路径是：

$$
\text{key} = \text{SHA256}(j\_invariant(E\_AB))[:16]
$$

加解密使用 AES-CBC（同 `IV`/`CT` 直接验证）。

```sage
def oracle(EA, phiAPB, phiAQB, EAB, pt):
    send_public_key(EA, phiAPB, phiAQB)
    shared = EAB.j_invariant()
    key = hashlib.sha256(str(shared).encode()).digest()[:16]
    ...
    return b"Good!" in ans
```

脚本还提供了 `sqrts.sage` 加速，说明 3-阶递归阶段需离散对数/平方根子例程支持，体现了题目从数学建模到交互 oracle 的典型两层链条。

## 方法总结

- 这题是“交互式同源映射 + 低位泄露 + Oracle 恢复”联合题，不应只看成 RSA 或普通 `isogeny` 解密。
- 关键复核清单：
  - `handout/chall.sage` 的 `j_ex()` 与 `encode/decode` 流程；
  - `admin/exploit/exp.sage` 的 `attack()` 三进制推进顺序；
  - 读写 `exp.sage` oracle 交互时 IV/CT 的编码是否完整（`hex`）。
- 按这个顺序可以完全复原两段 flag：先解第一段再驱动交互恢复第二段。
- 本次未重新运行依赖 SageMath 与交互服务的完整攻击；上述结果由 `chall.sage`、`s.sage`、`exp.sage`、两个官方 flag 文件及 README 交叉核对。
