---
type: tooling
tags: [blockchain, tooling, tools, environment]
skills: [ctf-blockchain]
---

# Blockchain Tooling

本页服务合约状态、交易执行、账户权限和链上证据。普通 dApp 的 HTTP 漏洞读取 [web-tooling.md](web-tooling.md)，独立签名数学读取 [crypto-tooling.md](crypto-tooling.md)。

## 平台与现存入口

RPC 查询、JSON 整理、字节与签名计算优先在 Windows 完成。Python 使用 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3），不使用 WSL `ctf-tools` 作为链工具环境。

| 入口 | 当前状态 | 适用工作 |
|---|---|---|
| PowerShell `Invoke-RestMethod`、`ConvertFrom-Json` | Windows 内置 | JSON-RPC 查询与响应检查 |
| Windows venv `requests` | `2.33.1` | 有界 RPC 客户端和题目交互脚本 |
| Windows venv `pycryptodome` / `cryptography` | `3.23.0` / `46.0.7` | Keccak、散列与基础密码操作；不代替完整 ABI/交易编码器 |
| Windows venv `gmpy2` / `sympy` / `z3-solver` | `2.3.0` / `1.14.0` / `4.16.0.0` | 合约约束与数学模型；详细用法归 Crypto 工具页 |
| Node.js | `C:/Program Files/nodejs/node.exe`，`24.13.0` | 已有 JavaScript 脚本；运行时存在不表示某链 SDK 已安装 |

Windows 与 WSL 当前 PATH 均未发现 `forge`、`cast`、`anvil`；本页没有配置好的 Foundry、Solana 或 Move 执行环境。需要编译器、链 SDK 或本地节点时，先按题目链与版本需求评估 Windows 方案，再确定最小依赖；不提供假定已安装的运行命令。

## 完整调用

查询题目 RPC 的链 ID，替换为已获授权的端点：

```pwsh
$chainRequest = @{
    jsonrpc = "2.0"
    id = 1
    method = "eth_chainId"
    params = @()
} | ConvertTo-Json -Compress
Invoke-RestMethod -Method Post -Uri "http://127.0.0.1:8545" -ContentType "application/json" -Body $chainRequest
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/analyze_chain.py"
& "C:/Program Files/nodejs/node.exe" "D:/题目路径/analyze_chain.mjs"
```

计算 Ethereum 方法选择器时使用 Keccak-256，不用 `hashlib.sha3_256` 替代：

```pwsh
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'from Crypto.Hash import keccak; h=keccak.new(digest_bits=256); h.update(b"transfer(address,uint256)"); print(h.hexdigest()[:8])'
```

只读调用先记录 chain ID、合约地址、区块高度与实际返回值。交易发送、账户状态修改或持续本地节点须在会话授权范围内执行；工具页示例本身不授予操作权限。

## 失败处理与下一跳

- RPC 不通或返回错误：核对 URL、认证、chain ID、节点方法支持与题目服务状态，先保留原始 JSON 错误，不通过重装 SDK 解决连接问题。
- ABI 解码不符：核对 selector、tuple/array 布局、返回数据和代理实现地址；转 [blockchain-smart-contract-exploitation.md](blockchain-smart-contract-exploitation.md)。
- 存储值异常：确认 slot、packing、delegatecall 上下文与代理布局；转 [delegatecall-storage-layout-corruption.md](delegatecall-storage-layout-corruption.md)。
- 跨链消息能验证却越权：核对字段绑定、域分隔和目标链；转 [bridge-message-field-binding.md](bridge-message-field-binding.md)。
- 只有字节码行为恢复时转 [reverse-tooling.md](reverse-tooling.md)；历史交易重建看 [blockchain-and-transaction-forensics.md](blockchain-and-transaction-forensics.md)。
- WSL 只用于实际依赖 Linux 的运行能力；任何 WSL 新增、升级、重装都须先说明用途、平台必要性、目标与依赖影响并取得明确同意。
