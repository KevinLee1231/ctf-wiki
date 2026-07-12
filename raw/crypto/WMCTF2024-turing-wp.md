# Turing

## 题目简述

题目模拟 Enigma。服务端随机选择 3 个转子、插线板、日密钥，并把含有固定 key 的明文加密多次。选手需要恢复足够接近的明文，通过相似度检查后获得 flag。

源码确认：服务端会先发 10 条随机 key 下的密文，再发一条目标密文，最后要求提交 plaintext；若相同位置比例超过 85%，返回 flag。

```python
if (count / len(s1)) > 0.85:
    return True
```

## 解题过程

### 关键机制

外链视频讲的是 Bombe 的“对角线板”思想。正文需要的核心信息是：根据已知明密文 crib 建图，插线板满足对称互换关系。如果某个候选转子顺序和日密钥导致某束线 26 根线全部被点亮，说明插线板约束冲突，可排除该候选；若每束最多只有少量候选，则可恢复 key 和大部分插线。

对角线板可理解为 $26\times26$ 布线矩阵，第 $i$ 束第 $j$ 根与第 $j$ 束第 $i$ 根相连，表达插线板互换的对称性。

参考 URL：https://www.bilibili.com/video/BV1PL4y1H77Z/?share_source=copy_web&vd_source=b2ff1691c43d0b58feed1e318e3afd1c

### 求解步骤

使用已知 crib：

```js
var plaintext = "THEWEATHERTODAYIS";
var ciphertext = "OXHAULBSHBSURPWKH";
var pos = 22;
```

枚举：

- 5 个转子中选 3 个并排列。
- 日密钥 $26^3$。
- 对角线板中选择一根线通电，DFS 传播约束。

判断函数的核心：

```js
function dfs(ci, cj) {
    if (bombMartix[ci][cj] !== 0) return;
    bombMartix[ci][cj] = 1;
    dfs(cj, ci);
    for (let i = 0; i < smlist[ci].length; i++) {
        const arr = smlist[ci][i].getValue(ci, cj);
        dfs(arr[0], arr[1]);
    }
}
```

若某一行被点亮 26 次，则该候选冲突：

```js
for (let i = 0; i < 26; i++) {
    let sum = 0;
    for (let j = 0; j < 26; j++) sum += bombMartix[i][j];
    if (sum == 26) return [false];
}
```

找到候选后恢复插线板，用对应 Enigma 解密服务端最后一条密文，提交恢复的明文。

## 方法总结

- 长 crib 使 Bombe 剪枝非常有效。
- 对角线板的作用是把插线板对称关系纳入传播，快速排除错误 key。
- 服务端只要求 85% 相似度，不必恢复所有插线。
