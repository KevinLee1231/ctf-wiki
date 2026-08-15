# BSidesAlgiers2025 - Snitchh

## 题目简述

题目只给出一个 `Setup` 合约。`whisper(string)` 会把字符串记入 `known`，同时触发 `NewWhisper(string)` 事件；`snitch(string)` 则计算输入的 Keccak-256，并与合约内固定的 `encrypted_flag` 比较。目标不是破解 Keccak，而是从链上历史事件中找回已经公开过的 whisper，再用固定哈希校验完整 flag。

关键代码为：

```solidity
event NewWhisper(string w);

function whisper(string memory w) external {
    require(!known[w], "I already know it");
    known[w] = true;
    emit NewWhisper(w);
}

function snitch(string memory flag) external {
    bytes32 encrypted = keccak256(abi.encodePacked(flag));
    require(encrypted == encrypted_flag, "you ain't a snitch");
    solved = true;
}
```

决定性机制是：事件日志不会因为合约没有 getter 而消失；历史 `NewWhisper` 的动态字符串仍可从交易回执日志中查询和 ABI 解码。

## 解题过程

先从实例提供的 RPC 与 `Setup` 地址查询从创世块开始的同名事件：

```bash
cast logs \
  --rpc-url "$RPC_URL" \
  --address "$SETUP_ADDRESS" \
  --from-block 0 \
  "NewWhisper(string)"
```

事件参数 `string` 是动态类型，因此日志 `data` 的布局依次为 32 字节偏移、32 字节长度、实际字符串和右侧补零。若命令行工具没有直接解出字符串，可按下面的最小代码处理每条 `data`：

```python
def decode_event_string(data_hex: str) -> str:
    raw = bytes.fromhex(data_hex.removeprefix("0x"))
    offset = int.from_bytes(raw[:32], "big")
    length = int.from_bytes(raw[offset:offset + 32], "big")
    return raw[offset + 32:offset + 32 + length].decode()
```

按区块号和日志序号排序后，两段有效内容为：

```text
wE_Ar3_juST_
W4RMING_uP
```

将其连续拼接并套入比赛格式，得到候选：

`shellmates{wE_Ar3_juST_W4RMING_uP}`

无需依赖在线实例也能做最终校验。对候选执行 Keccak-256，结果为：

```text
369645266477a21452a6b85f0440a139b7c8e49548c7db17939f7761b7e33349
```

它与源码中的 `encrypted_flag` 完全一致，因此可调用：

```bash
cast send \
  --rpc-url "$RPC_URL" \
  --private-key "$PLAYER_PRIVATE_KEY" \
  "$SETUP_ADDRESS" \
  "snitch(string)" \
  "shellmates{wE_Ar3_juST_W4RMING_uP}"
```

这一链路也与公开的 [Snitchh 解题记录](https://medium.com/@yanisderiche22/snitchh-c47e47383a2b) 相互印证；外部记录提供的关键信息已经在正文中完整还原，链接仅保留用于来源追溯。

## 方法总结

- Solidity 事件是永久链上证据，`private` 状态、缺少 getter 或前端不展示都不能隐藏已经写入日志的内容。
- 动态事件参数要按 ABI 的“偏移—长度—数据”布局解析，不能把整段 `data` 直接当 ASCII。
- 固定哈希在本题中是校验器而非需要逆向的密码原语；先恢复低熵、分片的日志明文，再对完整候选做 Keccak-256 校验即可闭环。
