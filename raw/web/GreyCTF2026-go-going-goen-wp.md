# Go Going Goen

## 题目简述

`Go Going Goen` 是共享 FastAPI/PostgreSQL 后端上的连续三阶段状态机。队伍 token 只用于换取会话 cookie；每个阶段的完成状态都按用户保存，必须按 Pinpoint、Queens、Tango 的顺序推进。决定性问题不是跨队数据读取，而是服务对状态、事务隔离与错误状态的错误建模。

- Stage 1 是带猜测配额的 Wordle 式接口；有效词子集由服务器 secret 的 HMAC 证明。
- Stage 2 在 `READ COMMITTED` 下逐行/逐列验证棋盘，再单独统计总皇后数。
- Stage 3 为每个提交先写一笔待处理账本，再在不同事务中按由网格哈希导出的锁顺序验证。

## 解题过程

### Stage 1：用配额后的响应区分有效词

`/api/v1/pinpoint/guess` 先检查单词是否属于 secret 选择出的 100 词子集，后检查五次猜测配额。配额耗尽后，两类输入的响应仍不同：不在子集返回 `400 invalid_guess`，在子集返回 `409 limit_reached`，且两者都不再消耗配额。因此将公开词表逐项请求即可恢复子集；前五次正常响应也同样视为命中。

```python
code, body = await submit_guess(client, word)
if code == 200:
    known_subset.add(word)
elif code == 409 and body.get("error") == "limit_reached":
    known_subset.add(word)
elif code == 400 and body.get("error") == "invalid_guess":
    pass
```

得到约 100 个候选后，每轮提交至多五个候选；达到上限就调用 `/api/v1/pinpoint/reset`，并遵从返回的 cooldown。正确词会返回 `result=correct` 并解锁 Stage 2。这里的漏洞是 membership oracle，不是尝试绕过限流。

### Stage 2：在验证末尾插入安全皇后

`/api/v2/queens/submit` 对每个 $i$ 依次查询第 $i$ 行和第 $i$ 列，最后再查询总数。在 `READ COMMITTED` 中每个 `SELECT` 看到新的已提交数据。若位置 $(r,c)$ 在行列检查完成后才插入，即 $\max(r,c)$ 小于当前验证进度，它不会破坏先前检查，却会被最后的总数统计看见。

官方 solver 先用空棋盘测出 submit 的实际墙钟时间，再启动一次 submit，在合适的时间比例后并发发送左上角 $38\times38=1444$ 个位置。服务接受 `queens` 数组且每请求最多 50 个，因此可拆成约 29 个 HTTP/2 批次。阈值为 1337，1444 个位置为网络抖动与迟到批次留下余量。

```python
submit_task = asyncio.create_task(client.post("/api/v2/queens/submit"))
await asyncio.sleep(submit_wall * fraction)
tasks = [
    asyncio.create_task(client.post("/api/v2/queens/add", json={"queens": batch}))
    for batch in batches
]
result = (await submit_task).json()
```

波次太早会导致逐行/列检查失败，太晚则总数不足；测量后的多组 `fraction` 重试可把提交落在“最后一个安全索引检查之后、总数统计之前”的窗口。返回 `result=win` 时进入 Stage 3。

### Stage 3：死锁留下可消费的待处理账本

锁顺序由网格 JSON 的 SHA-256 前八字节作为 PRNG seed，再随机排列六个 `row_balance:N` 锁：

```python
digest = hashlib.sha256(json.dumps(grid).encode()).digest()
rng = random.Random(int.from_bytes(digest[:8], "big"))
keys = [f"row_balance:{i}" for i in range(1, 7)]
rng.shuffle(keys)
```

先从一个有效基础网格随机改动少数格，离线枚举到两张网格：一张前三把锁集合是 `{1,2,3}`，另一张是 `{4,5,6}`。并发提交这对网格会在 `SELECT ... FOR UPDATE` 验证中以相反顺序争锁，PostgreSQL 终止其中一个事务并抛出 `ValidationDatabaseError`。

错误处理将 attempt 标为 `crashed`，但账本映射只显式处理 `accepted`、`rejected`、`validating`、`pending`；未知状态回退为 `PENDING`。预先创建的 \$100 entry 因而没有被回滚，成为可消费余额。每轮并发提交一对网格，累积十笔 orphan PENDING 到 \$1000 后调用 `/api/v3/tango/buy-flag`。成功返回：

```text
grey{re4D_c0mm1tTed_wIl1_n0t_s4v3_Y0u}
```

## 方法总结

- 核心技巧：分别利用配额响应 oracle、`READ COMMITTED` 的检查/使用竞态，以及事务失败后的账本状态回退错误。
- 识别信号：先做可区分但不消耗配额的检查；逐项验证后再聚合统计；在主事务之前写入资源、异常分支却未覆盖所有状态。
- 复用要点：并发题必须量化窗口。Stage 2 应先校准验证耗时并留足 surplus；Stage 3 则要从服务的锁排序算法构造真正会交错的输入，而不是盲目高并发请求。
