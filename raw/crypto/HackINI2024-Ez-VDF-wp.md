# Ez VDF

## 题目简述

服务端公布 RSA 模数 $N=PQ$ 和随机生成元 `generator`，计算极大指数下的证明：

$$
proof=generator^E\bmod N,\qquad E=2^{2^{256}}
$$

玩家需要提交另一个合法生成元及其证明。直接计算 `E` 或分解 `N` 都不可行，但模幂映射保留乘法结构，可以从服务端已给出的证明构造新证明。

## 解题过程

### 利用模幂同态

选择：

$$
usergenerator=generator^2\bmod N
$$

服务端随后会公开原生成元的 `proof`。目标生成元对应的正确证明为：

$$
usergenerator^E=(generator^2)^E=(generator^E)^2\equiv proof^2\pmod N
$$

因此完全不需要知道 $P$、$Q$ 或真实指数：

```python
from pwn import remote

io = remote("host", 8000)
N = int(io.recvline().decode().split("=")[1])
generator = int(io.recvline().decode().split(":")[1])

user_generator = pow(generator, 2, N)
io.sendlineafter(b"Give me your generator : ", str(user_generator).encode())

io.recvuntil(b"The proof is : ")
proof = int(io.recvline())
user_proof = pow(proof, 2, N)
io.sendlineafter(b"Give me your proof : ", str(user_proof).encode())
print(io.recvall().decode())
```

源码还要求新生成元位于 `(2, N - 1)`，且不等于原值及其相反数。随机实例中平方值几乎必然通过；若碰到极小概率退化实例，重新连接即可。flag 为：

```text
shellmates{g3n3r4t0r_3xp0n3nt14t10n_f0r_th3_w1n}
```

## 方法总结

- 核心技巧：利用 $f(x)=x^E\bmod N$ 的乘法同态，由已知 $f(g)$ 直接计算 $f(g^2)$。
- 识别信号：挑战允许选择与已知输入存在代数关系的新输入，并给出原输入的模幂结果时，应先尝试同态构造。
- 复用要点：VDF 的“大指数难算”并不自动保证不可伪造；协议还必须约束输入关系或使用绑定输入的证明系统。
