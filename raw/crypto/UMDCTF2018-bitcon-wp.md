# UMDCTF 2018 - Bitcon

## 题目简述

题目声称使用比特币区块链生成钱包私钥，并给出公钥 `2Ucke1nNG1iNym7kTCWcQrNgsi84CxhhL2B9`。仓库中的生成脚本表明，所谓随机数其实取自最新区块哈希的后 32 个十六进制字符；目标是找出对应区块并提交私钥字符串的 SHA-256。

## 解题过程

`random_secret_exponent()` 的核心逻辑如下：

```python
block_hash = requests.get(
    "https://blockchain.info/latestblock"
).json()["hash"]
secret = int(block_hash[32:].encode().hex(), 16)
```

区块链数据公开且不会提供不可预测的私密熵，因此只要从出题时间附近的区块高度向前枚举，就能为每个区块重建候选私钥，再按题目脚本的方式计算公钥并与目标比较。

官方 `solution.py` 找到的区块为：

```text
height = 516976
hash   = 00000000000000000048d20bddb91b3869f500e4ae94d4ff11afb3c8bd67360b
```

私钥取哈希的后半段：

```text
69f500e4ae94d4ff11afb3c8bd67360b
```

计算摘要：

```python
import hashlib

private_key = b"69f500e4ae94d4ff11afb3c8bd67360b"
print(hashlib.sha256(private_key).hexdigest())
```

输出为：

```text
43368da0a227a5b3d9c3432850a7678ac1dcf91045ad6eb71041cdfe6cf4e85e
```

该值与仓库 `README.md` 保存的 flag 摘要一致。

## 方法总结

公开区块哈希可以作为可复现的随机信标，却不能直接作为钱包私钥等秘密材料的熵源。遇到“使用公开数据生成密钥”的题目，应先定位种子来源和时间范围，再复现完整的密钥、公钥派生过程，而不是攻击椭圆曲线本身。
