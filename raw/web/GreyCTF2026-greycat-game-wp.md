# GreyCat Game

## 题目简述

这是一个 Node.js 跑酷小游戏。浏览器资源、障碍物和 localStorage 故意放有多段假 flag；真实 flag 不在静态文件中，而由服务端按 session 保存跑局进度。达到快速阶段后，`/api/ghost` 只返回 Base64 与 XOR 处理后的 `stamp`，客户端再把解出的片段绘制到背景上。

关键是保留同一 cookie session，按服务端认可的速度提交跑局样本，然后复现前端的解码逻辑；不需要从前端诱饵字符串中猜 flag。

## 解题过程

### 解锁服务端跑局状态

先请求 `/api/bootstrap` 创建/重置 session。`/api/run` 会拒绝时间、tick 或 score 倒退，也会限制每次增长：

```js
const plausible =
  deltaTick >= 0 && deltaScore >= 0 &&
  deltaTick <= Math.max(30, Math.floor(deltaMs / 6)) &&
  deltaScore <= Math.max(80, deltaTick * 8.5);
```

同一 session 需要至少 6 个被接受的样本，最后满足 `score >= 2250`、`tick >= 500`，并且从第一条样本开始经过至少 5 秒。可按约一秒的间隔递增提交，例如 `(tick, score)` 为 `(0,0)`、`(120,260)`、`(240,620)`、`(360,1180)`、`(480,1880)`、`(560,2290)`。这比直接伪造一个极大 score 更可靠，因为后者不通过 plausible 检查。

### 收集并还原 ghost 片段

解锁后连续请求 `/api/ghost?score=2290&lane=<0|1>`。服务按发放序号轮转六段 flag，并返回：

- `stamp`：XOR 后字节的 Base64；
- `traceId`：`ghost-<session-seed>-<one-based-index>`；
- `echo`：仅保留结构、把字母数字掩为 `.`。

前端 `decodeStamp` 的等价实现如下：

```python
import base64

def decode_stamp(stamp: str, trace_id: str) -> str:
    encoded = base64.b64decode(stamp)
    _, seed, one_based = trace_id.split("-")
    key_base = sum(map(ord, seed)) + (int(one_based) - 1) * 17
    plain = bytes(
        value ^ ((key_base + offset * 13) & 0xff)
        for offset, value in enumerate(encoded)
    )
    return plain.decode()
```

在 cookie 不变的情况下取六次 `stamp`，按服务发放顺序解码并拼接，得到：

```text
grey{th3_trex_rep1ac3d_by_a_gr3y_cat}
```

`/api/replay?view=debug` 可以在片段已经发放后导出同一 session 的编码 trace，适合作为收集完整性检查；它不会跳过 `/api/run` 的解锁条件。

## 方法总结

- 核心技巧：满足有状态 API 的真实性检查，再从前端恢复服务端不直接显示的 XOR 编码内容。
- 识别信号：静态资源有明显 bait string，但真正接口返回 session 关联的 opaque 字段；前端恰好包含这些字段的解码器。
- 复用要点：自动化时用 cookie jar 维持同一 session，并按服务端时间窗口发送合理的递增 telemetry。Base64 只是封装，`traceId` 中的 seed 与序号才是 XOR 密钥的一部分。
