# SU_Auth

## 题目简述

题目实现了一个简化的 CSIDH 私钥认证流程。私钥 `SUKEY` 是一组取值范围为 $[-3,3]$ 的小整数，服务端用它执行一串同源映射，并把所得 Montgomery 曲线参数作为 AES-ECB 密钥材料：

```python
ells = [*primes(3, 200), 269]
SUKEY = [randint(-3, 3) for _ in range(len(ells))]

def SuAuth(A, priv, LIMIT=True):
    if any(priv[i] == SUKEY[i] for i in range(len(ells))) and LIMIT:
        return "🙅SUKEY"
    E = EllipticCurve(GF(p), [0, A, 0, 1, 0])
    # 按 priv 中各指数执行对应的小素数度同源映射
    ...
    return E.montgomery_model().a2()
```

程序试图阻止选手直接提交真私钥，但 `any` 的短路行为泄露了第一个相等位置；同时，CSIDH 群作用中的指数存在周期，可以把恢复出的私钥平移到一个不与原向量逐项相等、但作用结果相同的等价密钥。

## 解题过程

### 利用短路时序逐项恢复私钥

测试第 $i$ 个分量时，其余位置统一填入不可能出现在 `SUKEY` 中的 `4`，只枚举当前位置的 $j\in[-3,3]$：

```python
def guess(i):
    for j in range(-3, 4):
        probe = [4] * i + [j] + [4] * (len(ells) - i - 1)
        io.sendline(str(probe).encode())
        start = time.time()
        io.recvuntil("🔑: ".encode())
        elapsed = time.time() - start
        if elapsed < 0.1:
            return j
```

当 $j=SUKEY_i$ 时，`any(...)` 在该位置得到真值并立即返回 `"🙅SUKEY"`；其余六次请求会继续执行昂贵的同源映射。因此响应时间形成稳定的二分类信号，可以恢复全部分量。

### 构造等价密钥

直接提交恢复出的 `SUKEY` 仍会触发禁止逻辑。题目使用的理想类作用满足相应的三阶关系，给每个指数统一加 $3$ 不改变最终群作用：

$$
\left(\prod_i \mathfrak l_i\right)^3=[1].
$$

因此构造

```python
equivalent_key = [e + 3 for e in recovered_key]
io.sendline(str(equivalent_key).encode())
```

新向量的每个分量都落在 $[0,6]$，不会与原来位于 $[-3,3]$ 的同一分量相等，但 `SuAuth(0, equivalent_key)` 与 `SuAuth(0, SUKEY)` 给出相同曲线参数。服务端据此解出 `SUDOOR`，执行明文 `./OPEN_THE_FLAG!` 并输出 flag。

## 方法总结

- 核心技巧：利用 `any` 的短路返回与昂贵后续计算形成时序侧信道，再利用群作用的指数周期构造等价私钥。
- 识别信号：秘密比较被放在短路表达式中，而且“不匹配”路径包含明显更重的密码运算时，应立即检查逐位置时序泄漏。
- 复用要点：恢复秘密并不一定意味着可以原样提交；还要分析校验逻辑允许的等价类。本题通过给所有指数加 $3$ 同时绕过逐项相等检查并保持认证结果。
