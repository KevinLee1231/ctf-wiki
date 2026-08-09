# The Last Honest Witness

## 题目简述

挑战合约要求调用 `claim` 时同时提交 Groth16 证明、5 个公开信号和 3 组权限 fragment。ZK 电路证明提交者知道满足 $pq=N$ 的素因子、RSA 明文 $m$ 以及能通向公开 Poseidon Merkle root 的路径；合约外层还检查 Franklin–Reiter 相关消息、secp256k1 小范围离散对数签名和 40 位截断 Keccak 碰撞。

题目把 $N,e,c$ 放在 `Setup` 的 storage slot 1、2、3 中，并通过部署事件的 indexed topic 发布 Merkle root。因而完整解法是先从链上恢复公开参数，再依次解决四个彼此独立的密码学子任务，最后组装一次交易。

## 解题过程

### 1. 读取链上参数

从 `Setup` 读取三个槽位：

```bash
cast rpc eth_getStorageAt "$SETUP" 0x1 latest --rpc-url "$RPC_URL"
cast rpc eth_getStorageAt "$SETUP" 0x2 latest --rpc-url "$RPC_URL"
cast rpc eth_getStorageAt "$SETUP" 0x3 latest --rpc-url "$RPC_URL"
```

分别得到 $N,e,c$。再计算事件签名 `WitnessRoot(bytes32)` 的 topic0，查询部署日志；日志 `topics[1]` 就是 indexed 的 `merkleRoot`。不要只抓事件 `data`，因为 indexed 参数不会出现在那里。

### 2. 分解 RSA 并生成 ZK witness

$p$ 与 $q$ 非常接近，使用 Fermat 分解：从 $a=\lceil\sqrt N\rceil$ 开始递增，直到 $a^2-N=b^2$ 为完全平方数，然后

$$
p=a-b,\qquad q=a+b.
$$

计算

$$
d=e^{-1}\bmod((p-1)(q-1)),\qquad m=c^d\bmod N.
$$

按附件 `poseidon_helper.js` 的规则由 $p,q,m$ 生成 commitment、leaf、Merkle path 和 nullifier，先在本地确认计算出的 root 与链上事件一致，再生成证明：

```bash
node poseidon_helper.js "$P" "$Q" "$M" --input input.json
npx snarkjs groth16 fullprove input.json \
  zk/LastHonestWitness.wasm \
  zk/LastHonestWitness_final.zkey \
  proof.json public.json
npx snarkjs zkey export soliditycalldata public.json proof.json
```

官方样例恢复出：

```text
p = 784493436055779473
q = 784493436055795861
m = 474401937379412746004845
```

### 3. 解三组 fragment

Franklin–Reiter 部分给出同一明文的相关消息密文。对

$$
f(x)=x^3-c_1,\qquad g(x)=(x+1337)^3-c_2
$$

在 $\mathbb Z_n[x]$ 上求最大公因式，得到一次因子 $x-m_0$，从而恢复 `franklinReiterPlaintext`。

ECC 部分的私钥满足 $x<2^{20}$。对公开 secp256k1 点运行 baby-step giant-step，恢复 $x$ 后按题目指定的 `messageHash` 生成规范 ECDSA 签名 $(v,r,s)$。官方实例中 $x=789123$。

碰撞部分只比较带固定域分隔前缀的 Keccak 摘要低 40 位。随机或顺序枚举不同的 32 位整数，把低 40 位映射到首次出现值；按生日界约 $2^{20}$ 级样本即可找到碰撞。官方样例为 `1656330` 与 `2582757`。

### 4. 提交 claim

将证明的 $A,B,C$、五个 public signals、Franklin–Reiter 明文、ECDSA 的 $v/r/s$ 和两个碰撞值按 ABI 顺序传给 `claim`。特别注意 snarkjs 输出的 $B$ 点坐标顺序与 Solidity verifier 的二维数组顺序，直接手抄时很容易颠倒。

官方 `exp/exp.py` 已自动化读取 storage、处理日志、构造 witness、生成 proof 和发送交易，适合作为最终复现入口。

## 方法总结

这道题并不是单一“ZK 黑盒”。最稳妥的做法是把链上公开数据、RSA witness、Groth16 证明和三个外层 fragment 分成独立可验证阶段：先检查 $pq=N$ 与 $m^e\equiv c\pmod N$，再检查本地 Poseidon root，随后分别验证相关消息、ECDSA 和碰撞，最后才组装 calldata。这样任何一个子任务失败都能在提交交易前定位，不会把 ABI 顺序问题误判为密码算法错误。
