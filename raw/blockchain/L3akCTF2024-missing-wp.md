# L3akCTF 2024 Missing Writeup

## 题目简述

仓库中的顶层 README 误写成了 `PySysMagic`，但源码实际是一道 Merkle Tree 恢复题。Solidity 合约保存一个不可变根哈希，并使用与 OpenZeppelin multiproof 相同的队列算法校验证明：

```solidity
bytes32 a = leafPos < leavesLen ? leaves[leafPos++] : hashes[hashPos++];
bytes32 b = proofFlags[i]
    ? (leafPos < leavesLen ? leaves[leafPos++] : hashes[hashPos++])
    : proof[proofPos++];
hashes[i] = commutativeKeccak256(a, b);
```

已知叶子由“位置字节 + 字符”组成。初始材料给出 18 个叶子及若干证明节点，缺少的字符可以通过枚举候选叶子和内部节点、与已知证明哈希比对来恢复。链上合约只是验证载体，决定性工作是理解其 Keccak Merkle 节点编码。

## 解题过程

一个叶子的明文为：

$$
\operatorname{leaf}(i,c)=\operatorname{Keccak256}(\operatorname{byte}(i)\mathbin{\|}\operatorname{UTF8}(c)).
$$

父节点先按 32 字节数值排序，再拼接哈希，因此树是可交换的：

$$
H(a,b)=\operatorname{Keccak256}(\min(a,b)\mathbin{\|}\max(a,b)).
$$

这意味着恢复字符时不需要猜左右方向。先把已知叶子按位置填入 flag：

```text
L3AK{M3rk?3_Tr33s_R_FuN_8ut_wh47_4r?_th3s3_s4?ts_f?r}
```

### 1. 枚举直接出现在 proof 中的叶子

位置范围是 $0\ldots52$，字符集限制为大小写字母与数字。对所有尚未出现的位置和候选字符计算叶子哈希，若结果等于某个未解释的 proof 节点，就能直接确定该字符：

```javascript
for (const position of missingPositions) {
  for (const character of alphabet) {
    const leaf = keccak(
      Uint8Array.from([position, character.charCodeAt(0)])
    );
    if (proofSet.has(leaf)) {
      recovered.push([position, character]);
    }
  }
}
```

### 2. 枚举二叶子分支

有些 proof 项不是叶子，而是两个未知叶子的父节点。把所有候选叶子两两组合并计算 $H(a,b)$，与剩余 proof 哈希比较，即可一次恢复两个字符。因为合约使用可交换哈希，每对候选只需计算一次。

### 3. 处理最后的四叶子子树

前两步后只剩位置 $9,35,45,50$，字符分别存在 `l/1`、`e/3`、`l/1`、`o/0` 的视觉歧义。枚举这 $2^4$ 种取值以及四个叶子的配对顺序，计算：

$$
H\bigl(H(l_0,l_1),H(l_2,l_3)\bigr),
$$

并与最后一个目标分支
`80787accc60ddad9832dbe33e0a73b81d0f4b7b5b8363af14e638671c20e6d27`
比较。唯一匹配为：

```text
position 9  = l
position 35 = 3
position 45 = l
position 50 = 0
```

完整结果是：

```text
L3AK{M3rkl3_Tr33s_R_FuN_8ut_wh47_4r3_th3s3_s4lts_f0r}
```

## 方法总结

- 先严格复现合约的字节编码与哈希顺序；把位置写成 ASCII 十进制文本、使用 NIST SHA-3，或漏掉节点排序都会得到完全不同的树。
- Merkle proof 中的 32 字节项可能是叶子，也可能是任意高度的子树根。应按“单叶 → 两叶分支 → 更高层小子树”逐层扩大枚举。
- 可交换节点哈希抹去了左右方向，降低了枚举量，但也可能保留多组排列歧义；最终必须用更高层 proof 节点或链上根哈希消歧。
