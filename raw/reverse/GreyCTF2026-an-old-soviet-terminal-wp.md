# An old soviet terminal

## 题目简述

远端服务允许提交最多 600 字节的 Troupe 程序。Troupe 是带信息流控制（IFC）的 actor 语言：flag 在 `transmitter.trp` 中以 `{topsecret}` 标签发送给 receiver；用户不能直接调用 `declassify` 或使用 `authority`，过滤器也会拒绝这些关键词。

但 receiver 预装了两个特权服务。`analysisService` 能返回 transmission 长度，也能比较 transmission 的一个字符与用户给出的候选；`logService` 会以自己的 authority 对收到内容执行 `declassify(content, authority, {})` 并将结果回传。于是看似只能接触高保密数据的布尔比较结果，被特权日志服务降级为公开值。核心工作是还原自定义语言的 actor/标签传播语义并构造 oracle，归为 reverse，而非普通网络服务漏洞。

## 解题过程

### 找到可用的高保密比较 oracle

关键服务逻辑可概括为：

```ml
(* analysisService *)
hn ("analyze", sender) =>
    send(sender, ("analysis",
        declassify_with_block(stringLength transmission, authority, {})))

hn ("compare", sender, idx, ch) =>
    send(sender, ("comparison", charAt transmission idx = ch))

(* logService *)
hn ("log", sender, content) =>
    let val sanitized = declassify(content, authority, {})
    in send(sender, ("logged", sanitized)) end
```

`analyze` 合法泄露长度；`compare` 的布尔结果带随 transmission 而来的高标签。玩家代码由 receiver 执行，因而能接收该结果；再把 result 作为 `log` 的 content 发给 `logService`，即可取回已经去标签化的 `true` 或 `false`。这不是绕过输入正则：正则只挡住用户直接使用特权 primitive，却错误信任了可被任意调用的特权服务。

### 按位置信息逐字符恢复

先发送 `("analyze", self())` 取得 $L$，再对每个索引 $i=0,\ldots,L-1$ 依次枚举有限字符集。每次尝试按下列消息序列进行：

```text
send(analysisService, ("compare", self(), i, candidate))
receive ("comparison", result)
send(logService, ("log", self(), result))
receive ("logged", public_result)
```

若 `public_result` 为真，就确定当前位置。官方 solver 使用大小写字母、数字、`{}`、`_`、`!`、`@`、`-` 作为候选集，并在 Troupe source 内递归连接字符。压缩后的 `retriever_code` 长 578 字节，满足 600 字节限制；提交时末尾再单独写 `EOF`。

```bash
python solve.py <host> <port> --retries 60 --timeout 150
```

由于服务端为 Troupe runtime 设置了并发槽、队列与短超时，官方 solver 对 `[BUSY]`、`[TIMEOUT]`、队列满等结果重试；这只处理基础设施不稳定性，不改变 oracle 的逻辑。

### 验证

按长度和逐字符 comparison oracle 还原出的传输为：

```text
grey{Th3_w4l1S_h4v3_E4r5}
```

同一字符串也被 `transmitter.trp` 以 `{topsecret}` 标签初始化，形成源码级的交叉验证。

## 方法总结

- 核心技巧：分析受 IFC 保护的 actor 系统时，审查所有拥有 authority 的服务是否会把高标签 predicate 或派生值回传到低域。
- 识别信号：服务禁止调用者直接 `declassify`，却提供“比较高保密对象”和“日志/审计”的特权接口时，应检查这两个接口能否组合成逐字符 oracle。
- 复用要点：先泄露长度再枚举有限字符集，将每轮 true/false 安全地转为公开判断；面对异步服务，区分可重试的队列/超时失败和 oracle 本身返回的否定结果。
