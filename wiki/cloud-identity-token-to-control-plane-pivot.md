---
type: technique
tags: [cloud-infra, identity, token, kubernetes, control-plane, technique]
skills: [ctf-cloud-infra, ctf-pentest]
raw:
  - ../raw/cloud-infra/UMDCTF2023-flamecamp-wp.md
  - ../raw/cloud-infra/WMCTF2022-java-wp.md
updated: 2026-07-27
---

# Cloud Identity Token to Control-Plane Pivot

## 适用场景

初始入口只是客户端配置、SSRF、环境变量或低权限服务，但其中可恢复云身份 token、service account 凭据或项目标识，并继续访问对象存储、Kubernetes API、Serverless 管理面或集群内服务。主线是身份到资源图的权限传播，不是单一 HTTP 漏洞。

## 识别信号

- 客户端包中包含 Firebase/cloud project 配置、公开注册端点或弱存储规则。
- SSRF 能读取 `file://`、`/proc/self/environ`、metadata endpoint 或 service account 文件。
- 环境中出现 bearer token、namespace、CA、API Server `:6443`、Pod/Service 名称。
- 控制面枚举出的内部服务继续暴露管理 API、任务提交或命令执行面。

## 最小证据

- 记录 token 的签发者、主体、audience、过期时间和实际权限，不把 API key 当身份凭据。
- 用最小只读请求验证 token 能访问哪个资源或控制面对象。
- 建立从入口到目标的资源图：身份、策略、对象、Pod、Service、内部应用和执行权限。
- 每次 pivot 保存响应状态和权限边界，区分网络不可达、认证失败和授权不足。

## 解法骨架

1. 从客户端配置、源码、环境、metadata 或挂载目录提取项目标识和身份材料。
2. 通过官方身份端点或当前工作负载身份换取短期 token，并解析 claims。
3. 以只读接口枚举对象、namespace、Pod、Service 和可达内部端口。
4. 根据权限选择最短下一跳：读敏感对象、调用控制面、提交任务或访问内部管理服务。
5. 对二段服务单独验证版本和输入编码，避免把控制面成功误认为最终执行成功。

## 关键变体

- Firebase API key 只定位项目，真正的授权取决于 identity token 和 Storage/Database rules。
- Kubernetes service account token 可能只读 namespace，也可能因 RBAC 误配获得跨 namespace 能力。
- SSRF 转发自定义 header 时可直接携带 bearer token；不转发时需要寻找 query、DNS 或内部代理替代。
- 控制面本身安全但集群内 Spark/Jenkins 等应用存在命令提交能力时，攻击链在应用层结束。

## 常见陷阱

- 看到云 API key 就直接称为密钥泄露，没有验证它能否代表用户或工作负载身份。
- 只读取 token，不解析 audience、namespace 和 RBAC，导致在无权限接口上盲试。
- 一开始就做大范围集群扫描，忽略 API Server 已提供的资源关系。
- 多层 URL 编码按经验叠加，没有按每个解析器逐层复算。

## 关联技巧

- [oauth-saml-cors-and-cicd.md](oauth-saml-cors-and-cicd.md)
- [pentest-attack-chains-and-tunneling.md](pentest-attack-chains-and-tunneling.md)
- [workflow-runner-internal-api-chain.md](workflow-runner-internal-api-chain.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)

## 来自 WP 的案例索引

本节只保留可复用识别信号，不替代原始题解正文。

| Raw WP | 可复用联系 |
|---|---|
| [WMCTF2022-6166lover-wp](../raw/cloud-infra/WMCTF2022-6166lover-wp.md) | 本题核心是从应用 RCE 转到云上身份利用。Python debug 路由能拿 shell，但在线容器里的 `/flag` 已在启动后删除，所以真正目标变成读取实例元数据、拿 RAM role 临时凭证、访问 ACR 并拉取题目镜像。 |

## 原始资料

- [UMDCTF2023-flamecamp-wp.md](../raw/cloud-infra/UMDCTF2023-flamecamp-wp.md)
- [WMCTF2022-java-wp.md](../raw/cloud-infra/WMCTF2022-java-wp.md)
