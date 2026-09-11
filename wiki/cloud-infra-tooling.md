---
type: tooling
tags: [cloud-infra, tooling, tools, environment]
skills: [ctf-cloud-infra]
---

# Cloud / Infra Tooling

本页服务 IAM、资源策略、metadata、Kubernetes 控制面、CI/CD 与制品信任问题。部署在云上的普通应用漏洞仍用 [web-tooling.md](web-tooling.md)。

## 平台与当前状态

JSON、YAML、JWT、HTTP 和制品文件检查默认在 Windows 完成，Python 固定使用 `D:/文档/新建文件夹/venv/Scripts/python.exe`（3.14.3）。分析目标运行 Linux 或 Kubernetes，不等于本地客户端必须在 WSL。

| 入口 | 当前状态 | 用途 |
|---|---|---|
| `Invoke-RestMethod`、`ConvertFrom-Json` | PowerShell 内置 | 已授权 API 的定点查询与结构化响应 |
| `C:/Windows/System32/curl.exe` | `8.21.0` | 原始 HTTP 请求、响应头与连接诊断 |
| Windows venv `requests` | `2.33.1` | 可复现的 HTTP/metadata 交互脚本 |
| Windows venv `PyYAML` | `6.0.3` | 用 `yaml.safe_load` 阅读策略、工作流和清单 |
| Windows venv `PyJWT` / `cryptography` | `2.12.1` / `46.0.7` | JWT 结构和签名语义；解析 payload 不等于验证 token |
| `C:/Windows/System32/OpenSSH/ssh.exe` | Windows 已有入口 | 已授权 SSH 连接及端口转发 |
| `/usr/bin/jq` | WSL APT `1.8.2-1` | 已有 Linux 工作流中的 JSON 筛选；普通 Windows JSON 分析无需切换平台 |

Windows 与 WSL 当前 PATH 均未发现 `docker`、`kubectl`、`aws`、`az`、`gcloud`、`terraform`。这表示没有可直接调用的这些 CLI，不代表题目云端也缺少它们。先用现有 API/文件分析能力；确需厂商 CLI、容器或 IaC 执行时，另行选择 Windows 兼容版本和最小安装范围。

## 完整调用

以下只读示例从 `pwsh` 发起；替换成当前题目的文件与授权端点：

```pwsh
Get-Content -LiteralPath "D:/题目路径/policy.json" -Raw | ConvertFrom-Json
& "D:/文档/新建文件夹/venv/Scripts/python.exe" -c 'from pathlib import Path; import yaml; print(yaml.safe_load(Path("D:/题目路径/workflow.yml").read_text(encoding="utf-8")))'
& "C:/Windows/System32/curl.exe" --max-time 10 --include "http://127.0.0.1:8080/health"
& "D:/文档/新建文件夹/venv/Scripts/python.exe" "D:/题目路径/analyze_policy.py"
```

已经处于 Linux 工件处理链时，可直接用系统工具，不套 Conda：

```pwsh
wsl -d kali-linux --exec /usr/bin/jq . "/mnt/d/题目路径/policy.json"
```

记录 principal、resource、action、条件和信任来源，再判断下一次请求。不要把一次 metadata 响应直接等同于可调用所有控制面 API，也不要把 token payload 里的 role 当作服务端已经授权。

## 失败处理与下一跳

- 401/403：核对凭据来源、受众、scope、过期时间、资源策略与签名要求；转 [cloud-identity-token-to-control-plane-pivot.md](cloud-identity-token-to-control-plane-pivot.md)。
- metadata 可读但调用失败：确认代理与网络路径、临时凭据完整性、目标区域和实际 principal，不盲目安装整套云工具。
- 制品内容被信任或加载：转 [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)；证据提取见 [forensics-tooling.md](forensics-tooling.md)，恶意载荷语义见 [malware-tooling.md](malware-tooling.md)。
- CI runner/internal API 越权：转 [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)；身份与协议边界看 [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)。
- Windows 缺包先评估兼容的新版本；仅当本地复现确需 Linux 能力时考虑 WSL。WSL 新增、升级或重装必须先说明用途、必要性、目标和依赖影响并取得明确同意。
