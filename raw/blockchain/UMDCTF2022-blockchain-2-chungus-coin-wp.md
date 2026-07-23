# UMDCTF2022 Blockchain 2 - Chungus Coin Writeup

## 题目简述

服务实现了一条简化的 PoW 区块链。玩家需要注册一个未被占用的节点名，读取当前链和待处理交易，挖出恰好延长当前链一个区块的新链，再提交给 `/nodes/update`。服务接受新链后把 flag 作为挖矿奖励返回。

PoW 条件为：前一区块 proof 为 $p_0$、新区块 proof 为 $p_1$ 时，

$$
\operatorname{SHA256}(\operatorname{str}(p_0)\Vert\operatorname{str}(p_1))
$$

的十六进制摘要必须以 `00000` 开头。

## 解题过程

先用一个不在 `blockchain.names` 中、也未被其他玩家注册的名称调用：

```http
POST /nodes/register
Content-Type: application/json

{"node_name":"unique-player-name"}
```

随后分别读取 `/chain` 和 `/pending_transactions`。新区块必须满足服务端 `valid_chain` 的全部检查：

- `index` 等于当前末块索引加一；
- `previous_hash` 是前一区块按键排序 JSON 的 SHA-256；
- 时间戳不小于前一区块；
- proof 满足五个前导十六进制零；
- `transactions` 与服务端当前待处理交易列表完全相同。

最小挖矿逻辑如下：

```python
import hashlib

def valid_proof(last_proof, proof):
    guess = f"{last_proof}{proof}".encode()
    return hashlib.sha256(guess).hexdigest().startswith("00000")

def proof_of_work(last_proof):
    proof = 0
    while not valid_proof(last_proof, proof):
        proof += 1
    return proof
```

按服务端相同的序列化方式计算前块哈希：

```python
previous_hash = hashlib.sha256(
    json.dumps(chain[-1], sort_keys=True).encode()
).hexdigest()
```

构造新区块并附加到刚读取的链：

```python
block = {
    "index": chain[-1]["index"] + 1,
    "timestamp": time(),
    "transactions": pending_transactions,
    "proof": proof_of_work(chain[-1]["proof"]),
    "previous_hash": previous_hash,
}
chain.append(block)
```

最后提交：

```python
requests.post(
    f"{base}/nodes/update",
    json={"chain": chain, "length": len(chain), "name": name},
)
```

验证通过后响应中的 `reward` 给出：

```text
UMDCTF{Chungus_Th4nk5_y0u_f0r_y0ur_bl0ckch41n_s3rv!c3}
```

## 方法总结

本题不需要伪造交易：服务端要求新区块交易与当前 pending 列表完全一致。关键是复制当前状态、按服务端规则挖出一个有效 proof，并保持链只增长一块。抓取 pending 后应立即提交，避免状态变化导致精确相等检查失败；JSON 哈希也必须使用 `sort_keys=True`，否则 `previous_hash` 不一致。
