---
type: technique
tags: [web, path-traversal, double-url-encoding, ssrf, upload, nodejs, require-rce, aes]
skills: [ctf-web, ctf-cloud-infra]
raw:
  - ../raw/web/Spirit2026-5-artifact-relay-wp.md
updated: 2026-05-21
---

# Artifact Trust SSRF to Node require RCE

## 适用场景

上传 artifact 后，Web 服务提供文件查看、扫描或报告接口。路径过滤先检查字面量再解码，导致双重 URL 编码穿越；穿越可读到加密密钥和 runner token；公开服务又提供内部 runner 转发入口，攻击者把恶意 artifact 标记为 trusted 后触发 runner `require()` 用户文件，最终 RCE。

## 识别信号

- 文件读取接口参数类似 `name`、`path`、`file`，过滤 `..` 发生在 URL decode 之前。
- 上传产物会被解包、扫描、生成报告，报告接口能读取 artifact 内部或相邻路径文件。
- 源码中存在硬编码 AES key / IV，或 token 文件以 `.enc` 形式保存在 data 目录。
- 公开路由 `/internal/runner/*` 会转发给内部 runner。
- trusted scan、plugin scan、rules scan 会 `require()`、`import()` 或执行 artifact 中的规则文件。

## 最小证据

- 已用双重编码路径读取到 artifact 根目录之外的文件。
- 已通过泄露的 key 解密 runner token，或找到能伪造 runner 认证的材料。
- 已确认 trusted artifact 中的规则文件会在 runner 权限下执行。

## 解法骨架

1. 上传无害 artifact，拿到合法 artifact id。
2. 测试 `%252e%252e%252f` 是否能在二次解码后变成 `../`。
3. 读取源码或配置文件，找 crypto key、runner token 路径、内部路由和 trusted scan 逻辑。
4. 读取加密 token 并离线解密。
5. 构造恶意 artifact，保持 `.relay/rules.js` 位于解包根目录。
6. 通过 SSRF / internal relay 标记 trusted，再触发 trusted scan。
7. 从扫描报告或 findings 读取命令输出。

## 关键变体

- **双重 URL 解码。** `%252e` → `%2e` → `.`。
- **token 不是明文。** 常见链路是读 crypto 源码，再读 `.enc` 文件，最后 AES-CBC 解密 token。
- **trusted 状态是执行门槛。** 只上传恶意规则不一定执行。
- **Node require 执行模型。** `require('.relay/rules.js')` 会执行模块顶层代码并调用导出函数。

## 常见陷阱

- 路径穿越 payload 编码层数不对。
- artifact 打包路径错误，`.relay/` 不在解包根目录。
- `/root/flag` 和 `/root/flag.txt` 路径不一致，需要从源码确认。
- SSRF 桥梁需要 runner token；没有 token 往往只会 401。

## 关联技巧

- [sqli-filter-and-oracle-family.md](sqli-filter-and-oracle-family.md)
- [path-traversal-ssrf-upload-and-rsc.md](path-traversal-ssrf-upload-and-rsc.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [node-and-prototype.md](node-and-prototype.md)
- [php-java-python-deserialization.md](php-java-python-deserialization.md)

## 来自 WP 的案例索引

| Raw WP | 可复用联系 |
|---|---|
| [D3CTF2021-real-cloud-serverless-wp](../raw/cloud-infra/D3CTF2021-real-cloud-serverless-wp.md) | 云函数/Serverless/Kubernetes 内部 API 链，先固定 SSRF 到控制平面的可达边界。 |
| [D3CTF2021-real-cloud-storage-wp](../raw/web/D3CTF2021-real-cloud-storage-wp.md) | 云存储/内部 metadata 或可信服务接口可被 SSRF 触达，先确认凭据和对象读取路径。 |
| [D3CTF2021-shellgen-wp](../raw/web/D3CTF2021-shellgen-wp.md) | token 路径穿越写入模板目录形成 Jinja SSTI，再连 rootless Docker socket 控宿主用户权限。 |
| [HGAME2026-easyuu-wp](../raw/web/HGAME2026-easyuu-wp.md) | 可控目录列表找到更新包源码，上传后门 `path1` 可覆盖 `./update/easyuu`，再借自更新机制执行自定义程序。 |

## 原始资料

- [Spirit2026-5-artifact-relay-wp.md](../raw/web/Spirit2026-5-artifact-relay-wp.md)
