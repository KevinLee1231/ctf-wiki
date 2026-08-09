# Transit Privilege

## 题目简述

网页表面只有普通登录，但附件 `edge-agent-client-0.1.0.jar` 暴露了 `/proxy` WebSocket 协议及固定 HMAC key。完整利用分三段：伪造 `cap.sync` 协商创建 OPERATOR 凭据；滥用 workspace 的 retained routing 把自己提升为 ADMIN；从管理员 backup 下载服务端 JAR，构造 `InventoryCursorEntry` 反序列化对象，利用 Unicode char 到 byte 的截断差异读取 `/flag`。

这不是单点漏洞，任何一段都不能省略：代理协议提供初始账户，工作流提供权限，源码备份才揭示最终文件读取 gadget。

## 解题过程

### 1. 通过 /proxy 创建 OPERATOR

客户端 JAR 中的固定 key 为：

```text
6Ziy5ZCb5a2Q5LiN5aao5bCP5Lq6ISEh
```

连接 `ws(s)://host/proxy`，发送 `HELLO` 取得 `nonce`、`profile`、`routeEpoch`。AUTH 使用 HMAC-MD5，规范串为：

```text
identifier=Transit&authVersion=1&timestamp=<ms>&nonce=<nonce>
```

AUTH 后用 `DESCRIBE` 查询 `edge.capability`，取得 `cap.sync` 的动态 routeId。第一次 CALL 提交随机 `identity`、拟创建的 `principal` 和满足强度要求的 `secret`，服务返回 `proof_required` 与 ticket。证明为固定 key 上的 HMAC-SHA256 前 24 个十六进制字符：

```text
proof|cap.sync|nonce|routeEpoch|profile|identity|principal|ticket
```

切换到 ticket scope，带原字段、ticket、proof 再 CALL；请求 body 先做排序紧凑 JSON 的 SHA-256，CALL 自身再按 JAR 的 canonical string 做 HMAC-MD5。成功响应出现 `scope=operator-console`，新 `principal/secret` 可登录网页，角色为 OPERATOR。

### 2. 利用 workspace routing 提权

访问 `/api/workspace/bootstrap` 获取当前会话动态生成的 `draftActionId`、`previewActionId`、`submitActionId` 和 `advanceActionId`，不要硬编码旧 UUID。通过统一入口 `/api/workspace/action` 创建 draft、preview 后，提交：

```json
{
  "policyRef": "desk-default",
  "routing": {
    "mode": "retain",
    "handoff": "owner"
  }
}
```

retained resolver 把审批人重新绑定为当前 owner。再用 `advanceActionId` 推进自己的记录到 `APPROVED`，`/admin/me` 即显示 ADMIN。

### 3. 从 backup 取得服务端实现

管理员 `/api/backup/list` 中存在 `support-source`，profile 为 `server-source`。调用 create/fetch 下载快照，其中的 `ops-console-demo-0.1.0.jar` 显示 `/admin/maintenance/reconcile` 会从上传 zip 读取 `inventory.dat` 并用原生 `ObjectInputStream` 反序列化。

类 `ctf.sctf.ops.maintenance.InventoryCursorEntry` 的 `readObject()` 会调用 `ProbeSandbox.renderSnapshot(name, profile, cursor)`，结果写入可查询的 maintenance report。构造同 FQCN、`serialVersionUID=7319048271L`、字段为 `name/cursor/profile` 的对象，设置 `profile=merge` 后序列化进合法 bundle。

### 4. 利用表示差异读取 flag

过滤器检查原 Java String 中是否含 `../` 等穿越形式；但后续 `LegacyCursorAdapter.flattenToken()` 对每个 UTF-16 char 执行 `(byte) codeUnit`，再按 ISO-8859-1 重建路径。以下汉字的低 8 位分别是目标 ASCII：

```text
售 -> 0x2e '.'    启 -> 0x2f '/'
书 -> 0x66 'f'    公 -> 0x6c 'l'
卡 -> 0x61 'a'    剧 -> 0x67 'g'
```

所以可见 cursor：

```text
售售启售售启售售启书公卡剧
```

能通过前置过滤，却在 materialize 后变成 `../../../flag`。把对象放入 `inventory.dat`、附上合法 `manifest.json`，Base64 后提交 `/admin/maintenance/reconcile`，再按返回的 importId 查询 reports，得到：

```text
SCTF{Tr4ns1t_Pr0b3_4107_M@sTer}
```

## 方法总结

本题的共同缺陷是不同层对同一身份或字符串采用了不一致解释：paymaster 式代理能力被映射成网页登录账户，routing alias 把 owner 当审批人，路径过滤看 Unicode char，而文件层只看低 8 位。解题时应以客户端和服务端 JAR 为协议权威，动态读取 nonce、routeId、actionId 和 importId；正文中的临时示例值不能代替实际响应。最终的 Unicode 路径也必须在本地执行 `(byte) char` 后确认物化结果，再发送反序列化包。
