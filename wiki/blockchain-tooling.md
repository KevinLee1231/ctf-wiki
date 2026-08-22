---
type: tooling
tags: [blockchain, tooling, tools, environment]
skills: [ctf-blockchain]
---

# Blockchain Tooling

本页是 `ctf-blockchain` 方向本机工具信息的唯一权威来源，维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-blockchain/SKILL.md` 只说明何时选择某类工具或知识页，不复制本页细节。

本页只描述当前真实状态；实际环境与本文不一致时，直接修正文中现状，不在本页累积核验记录或旧版本历史。

## 工具选择边界

- 未确认链、chain ID、RPC 和题目账户前，只做只读查询，不发送交易、不部署合约、不使用真实私钥。
- EVM 先区分“只读 RPC 足够”还是确实需要 Foundry、本地 fork、编译器或 SDK；Solana/Sui 等非 EVM 链必须按账户模型选择专属 CLI。
- 普通 dApp HTTP、前端和认证漏洞转 Web；纯签名、ZK 或数论缺陷转 Crypto，不因为出现交易字段就强行使用链工具。

## 完整调用约定

当前可用基线是 WSL 系统 `curl` 与 `jq`。所有命令从 `pwsh` 发起：

```pwsh
$rpc = "https://ctf-rpc.example"
$body = '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}'
wsl /usr/bin/curl -sS -H "Content-Type: application/json" --data-raw $body $rpc

wsl /usr/bin/jq . /path/to/transaction-receipt.json
```

RPC URL、方法和参数必须来自题目证据。向远端发送状态变更交易前，先用本地模拟、`eth_call` 或题目提供的 simulation 接口验证。

## 当前状态与路径

| 工具 | 当前状态/版本 | 路径或环境 | 何时使用 | 完整调用 |
|---|---|---|---|---|
| `curl` | 可用，8.21.0 | `/usr/bin/curl`，WSL system | 原始 JSON-RPC、REST 或 explorer API 的只读请求 | 见上方 RPC 示例 |
| `jq` | 可用，1.8.2 | `/usr/bin/jq`，WSL system | 保存后的 transaction、receipt、trace、ABI 或 account JSON | `wsl /usr/bin/jq . /path/to/input.json` |
| `cast` / `forge` / `anvil` | 未发现 | WSL/Windows `PATH` 与 `/home/kali` 常见路径均未发现 | EVM calldata、storage、trace、本地链和 Solidity 测试 | 当前无可执行命令；不要假设 Foundry 已安装 |
| `solc` | 未发现 | WSL/Windows `PATH` 与 `/home/kali` 未发现 | 编译器版本、ABI、bytecode 或 IR 对照 | 当前无可执行命令 |
| `web3.py` / `eth-account` / `py-solc-x` | `ctf-tools` 未安装 | WSL `ctf-tools` | Python 化 ABI、签名、交易和编译流程 | 当前不可导入；不要把普通 `requests` 当作 SDK |
| Solana / Anchor SDK | `solana`、`anchorpy` 未安装；对应 CLI 未发现 | WSL/Windows | Solana account/PDA/instruction 语义 | 当前无可执行命令 |
| Sui 等链专属 CLI | 未发现 | WSL/Windows | resource/capability/object 模型决定解法时 | 当前无可执行命令 |

## 失败处理

- `curl` 返回链不匹配、方法不存在或权限错误：先保存响应，核对 chain ID、RPC 网络、方法命名和题目账户，不立刻安装整套工具链。
- 只读 RPC 无法给出 trace/storage 解读且题目明确是 EVM：再评估安装 Foundry 或 `solc`；安装会改变环境，执行前按全局授权规则确认。
- 需要 Python SDK 但 `ctf-tools` 缺包：优先为题目创建独立环境或工具层，不直接污染通用 CTF 环境；环境确定后把版本、路径和完整命令更新到本页。
- 发现决定性障碍是 HTTP、钱包客户端二进制或密码数学时，分别转 [web-tooling.md](web-tooling.md)、[reverse-tooling.md](reverse-tooling.md) 或 [crypto-tooling.md](crypto-tooling.md)。

## 关联知识页

- [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md)
- [delegatecall-storage-layout-corruption.md](delegatecall-storage-layout-corruption.md)
- [bridge-message-field-binding.md](bridge-message-field-binding.md)
- [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)
