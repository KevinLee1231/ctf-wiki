---
type: tooling
tags: [cloud-infra, tooling, tools, environment]
skills: [ctf-cloud-infra]
---

# Cloud / Infra Tooling

本页是 `ctf-cloud-infra` 方向本机工具信息的唯一权威来源，维护当前安装状态、版本、路径、环境、完整调用、适用边界和失败处理。`ctf-cloud-infra/SKILL.md` 只说明何时选择某类工具或知识页，不复制本页细节。

本页只描述当前真实状态；实际环境与本文不一致时，直接修正文中现状，不在本页累积核验记录或旧版本历史。

## 工具选择边界

- 先确认厂商、account/project、region、cluster/context、namespace 和题目凭据，再选择 CLI；不读取或复用个人机器上的环境凭据作为题目默认值。
- 先用保存的响应、token claim、策略文档和最小只读 API 建 identity → action → resource → trust 图，再决定是否需要云 CLI、Kubernetes、registry 或 IaC 工具。
- 创建/删除资源、修改 IAM/RBAC、触发 pipeline、部署 workload 或扩大枚举范围前，必须再次确认题目范围和身份上下文。

## 完整调用约定

当前可用基线是 WSL 系统 `curl` 与 `jq`。所有命令从 `pwsh` 发起：

```pwsh
wsl /usr/bin/curl -sS -D /tmp/headers.txt https://ctf-control-plane.example/api/status
wsl /usr/bin/jq . /path/to/policy-or-token-claims.json
```

如果后续安装某个厂商 CLI，必须先在本页记录其当前绝对路径、版本和所需 context，再从 `pwsh` 以该绝对路径调用；不要依赖未知的默认 profile。

## 当前状态与路径

| 工具 | 当前状态/版本 | 路径或环境 | 何时使用 | 完整调用 |
|---|---|---|---|---|
| `curl` | 可用，8.21.0 | `/usr/bin/curl`，WSL system | 元数据、OIDC、控制面 API 和最小只读请求 | `wsl /usr/bin/curl -sS URL` |
| `jq` | 可用，1.8.2 | `/usr/bin/jq`，WSL system | policy、token claim、audit event 与 API JSON | `wsl /usr/bin/jq . /path/to/input.json` |
| AWS CLI | 未发现 | WSL/Windows `PATH` 与 `/home/kali` 未发现 | AWS identity、STS、IAM、S3、Lambda 等 | 当前无可执行命令 |
| Google Cloud CLI | 未发现 | WSL/Windows `PATH` | GCP identity、IAM、project 与资源图 | 当前无可执行命令 |
| Azure CLI | 未发现 | WSL/Windows `PATH` | Azure tenant、subscription、role 与资源图 | 当前无可执行命令 |
| `kubectl` | 未发现 | WSL/Windows `PATH` 与 `/home/kali` 未发现 | context、namespace、RBAC、service account、secret 与 workload | 当前无可执行命令 |
| `docker` / `skopeo` | 未发现 | WSL/Windows `PATH`；常见 Docker Desktop CLI 路径也不存在 | image、manifest、layer、registry 与签名检查 | 当前无可执行命令 |
| `terraform` | 未发现 | WSL/Windows `PATH` 与 `/home/kali` 未发现 | plan、state、provider、variable 与 IaC 信任边界 | 当前无可执行命令 |
| Python 云 SDK | `boto3`、`google-cloud-storage`、`azure-identity`、`kubernetes`、`docker`、`python-terraform` 均未装 | WSL `ctf-tools` | 只有需要脚本化厂商 API 时才考虑 | 当前不可导入 |

## 失败处理

- 没有厂商 CLI：先用题目给出的 JSON、HTTP 响应、token 和策略文本完成静态权限图；不要为了题型猜测一次性安装全部云工具。
- CLI 报认证或 context 错误：先记录当前 account/project/region/cluster/namespace 和错误响应；不要尝试个人 profile、真实云账号或扩大枚举。
- 镜像/registry 是证据恢复问题时转 [forensics-tooling.md](forensics-tooling.md)；恶意依赖或 C2 行为主导时转 [malware-tooling.md](malware-tooling.md)。
- 题目确认需要缺失工具后，安装应进入独立、可追踪的工具层；安装和验证完成后再把版本、路径与完整命令写回本页。

## 关联知识页

- [cloud-identity-token-to-control-plane-pivot.md](cloud-identity-token-to-control-plane-pivot.md)
- [artifact-trust-ssrf-to-node-require-rce.md](artifact-trust-ssrf-to-node-require-rce.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)
- [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)
