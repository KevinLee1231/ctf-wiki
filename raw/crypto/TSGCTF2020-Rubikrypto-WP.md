# TSGCTF2020 Rubikrypto WP

## 题目简述

题目在三阶魔方群上实现 ElGamal。flag 被当作大整数，以整个魔方群阶

$$
|G|=43252003274489856000
$$

为基分块，每块映射成一个合法魔方状态。对每个明文状态 $m$，程序随机选择群元素 $g$ 和指数 $x,y$，公开：

$$
h=g^x,
\qquad
c_1=g^y,
\qquad
c_2=m h^y.
$$

表面上需要在巨大群中求离散对数，但任意单个三阶魔方状态的元素阶都不超过 1260。于是只需枚举很小的循环子群，就能恢复 $x$ 在 $\operatorname{ord}(g)$ 下的值并解密。

## 解题过程

附件把每个 $g,h,c_1,c_2$ 写成一串魔方转动记号。用题目相同的 `cubejs` 解析后，从单位元开始反复右乘 $g$：

```javascript
const inverseH = Cube.inverse(h);
let x = 0;
const current = new Cube();

while (true) {
  x++;
  current.move(g);
  if (current.clone().move(inverseH).isSolved()) {
    break;
  }
}
```

循环在至多 1260 次内找到 $g^x=h$。即使服务端原始随机指数远大于 1260，得到的只是它模 $\operatorname{ord}(g)$ 的代表元；这已经足够，因为：

$$
c_1^x=(g^y)^x=(g^x)^y=h^y.
$$

因此明文状态为：

$$
m=c_2\left(c_1^x\right)^{-1}.
$$

对应实现：

```javascript
const c1Cube = new Cube().move(c1);
const c2Cube = new Cube().move(c2);
const shared = pow(c1Cube, BigInt(x));
const messageCube = c2Cube.move(shared.solve());
cubes.push(messageCube);
```

`solve()` 给出把 `shared` 还原到单位元的转动序列，也就是乘其逆元。对输出中的 10 组密文分别执行上述步骤，再调用附件的 `cubes2buffer`：它先把每个魔方状态逆映射为 $[0,|G|)$ 内的整数位，然后按基 $|G|$ 从高到低拼回原始大整数。

最终得到：

```text
TSGCTF{M4ybe_this_1s_r3as0n_why_w3_don't_h4ve_Rubik'5-Cube-bas3d_crypt0sy5tem...}
```

也可以分解群阶后使用 Pohlig–Hellman，但本题中单个元素的最大可能阶只有 1260，直接枚举明显更短。

## 方法总结

密码系统工作在“大群”并不意味着每个生成元都产生大阶子群。这里随机魔方状态的循环子群极小，离散对数最多枚举 1260 次，ElGamal 的核心困难完全消失。使用有限群构造密码协议时，必须选择并验证已知大素数阶子群的生成元；只公开整体群阶、再随机取任意元素并不能保证离散对数安全。
