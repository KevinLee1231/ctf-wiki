# Recovery Pod

## 题目简述

题目暴露一个受限的 TCP 调试终端，只允许 `help`、`cat <path>`、`gitcat <sha1>` 和 `quit`，不提供 `ls`、`find`、`git log` 或命令执行。挂载目录本身是一个 Git 工作区：`cat` 可以读取已知路径，`gitcat` 则把指定 loose object 的 zlib 压缩原文做 Base64 后返回。

恢复 flag 需要关联三类 Git 证据：普通提交中的 `scripts/recover_snapshot.py` 给出循环异或算法，stash 的 untracked-files parent 保存密文 `runtime_cache/bundle.txt`，`refs/notes/commits` 保存对应 key。核心不是突破终端限制，而是在不能列目录、不能运行 Git 命令的条件下手工遍历引用和对象图。

## 解题过程

### 从引用和 reflog 建立对象入口

连接服务后先读取 Git 的稳定入口：

```text
debug> cat .git/HEAD
ref: refs/heads/main

debug> cat .git/refs/heads/main
94fa9139f6fbfeb19b47ef746b90b181b06902e1

debug> cat .git/logs/HEAD
...
9563e3180e2991faec7d904497594aeb60794b35 94fa9139f6fbfeb19b47ef746b90b181b06902e1 ... add snapshot recovery helper
```

`HEAD` 给出当前引用，分支 ref 给出最新 commit，reflog 则提供即使无法执行 `git log` 也能利用的历史 SHA-1。Git loose object 通常位于 `.git/objects/<前两位>/<后 38 位>`，内容经 zlib 压缩；服务的 `gitcat` 先把这段压缩数据 Base64 编码，所以客户端必须先 Base64 解码，再 zlib 解压。

解压后的统一格式为 `<type> <size>\0<body>`。commit、blob 和 tag 的正文可直接显示；tree 正文则重复以下二进制记录：

```text
<mode><space><name><NUL><20-byte raw object id>
```

下面的辅助程序可解析服务逐行返回的对象：

```python
#!/usr/bin/env python3
import base64
import sys
import zlib


for line in sys.stdin:
    line = line.strip()
    if not line:
        continue

    obj = zlib.decompress(base64.b64decode(line))
    header, body = obj.split(b"\x00", 1)
    obj_type, size = header.decode().split(" ", 1)
    print(f"type={obj_type} size={size}")

    if obj_type in {"commit", "blob", "tag"}:
        print(body.decode("utf-8", "replace"))
    elif obj_type == "tree":
        pos = 0
        while pos < len(body):
            space = body.find(b" ", pos)
            nul = body.find(b"\x00", space)
            mode = body[pos:space].decode()
            name = body[space + 1:nul].decode("utf-8", "replace")
            object_id = body[nul + 1:nul + 21].hex()
            print(mode, object_id, name)
            pos = nul + 21
    print("---")
```

从最新 commit 展开其 tree，可以看到 `scripts` 子树以及 `recover_snapshot.py`；得知路径后也可直接执行 `cat scripts/recover_snapshot.py`。脚本的核心逻辑是对 Base64 解码后的密文做循环异或：

```python
def xor_bytes(data: bytes, key: bytes) -> bytes:
    return bytes(data[i] ^ key[i % len(key)] for i in range(len(data)))
```

### 从 stash 的 untracked parent 恢复密文

读取 `.git/refs/stash` 获得 stash commit，再通过 `gitcat` 解析。该 commit 有三个 parent：基准提交、索引快照以及 `git stash -u` 为未跟踪文件单独生成的 commit。第三个 parent 的提交说明为：

```text
untracked files on main: 94fa913 add snapshot recovery helper
```

继续展开它的 tree：未跟踪文件 commit 的根 tree 为 `cf44dbd951b57faba70670be87d572dc8a71ca68`，其中 `runtime_cache` 目录再指向子 tree `d12a5000aca8b912cb9f78c1667ea93453ee727a`，最终路径为：

```text
runtime_cache/                    -> tree d12a5000aca8b912cb9f78c1667ea93453ee727a
runtime_cache/bundle.txt          -> blob e20a32526011d11a9f4f98baab3a0ced59f56e23
```

blob 内容给出密文：

```text
snapshot_kind=runtime-cache
cipher_b64=VVEPWnlKVVYPWgICV1EVVVRaVBpaCVcGHVYFAwEZUgIJClJSUVJUAV0HHg==
```

这里不能只检查 stash 的主 tree；`-u` 保存的 untracked 文件位于额外 parent，忽略多父提交会漏掉真正的载荷。

### 从 Git notes 恢复 key 并解密

读取 notes 引用：

```text
debug> cat .git/refs/notes/commits
<notes_commit_sha1>
```

展开该 commit 和 tree，可见一个以当前提交 `94fa9139...` 命名的 notes 条目；对应 blob 为：

```text
cache snapshot key: 88a3516b8cc5df87d9d791d70e5b04ed
```

现在恢复算法、密文和 key 已全部齐备：

```bash
python3 recover_snapshot.py \
  88a3516b8cc5df87d9d791d70e5b04ed \
  VVEPWnlKVVYPWgICV1EVVVRaVBpaCVcGHVYFAwEZUgIJClJSUVJUAV0HHg==
```

输出为：

```text
miniL{c479a737-b0c0-c831-30a1-7f123adcbced}
```

题目启动脚本能反向验证整条链：它取 flag 的 SHA3-512 十六进制摘要前 32 字符作为 key，用该 key 循环异或 flag，把结果写入 untracked 的 `runtime_cache/bundle.txt` 后执行 `stash push -u`；同时把 key 写入当前 commit 的 Git note，最后删除工作区缓存和相关 reflog。上述三处恢复结果与该生成逻辑完全一致。

## 方法总结

- 核心技巧：在只允许按已知路径读文件和按 SHA-1 读对象的环境中，从 `HEAD`、refs、reflog 建立入口，手工解析 commit/tree/blob，再关联 stash 与 Git notes。
- 识别信号：工作区看不到目标文件，但 `.git` 可读，且题面强调恢复、快照或运维痕迹；应检查 `refs/stash`、`refs/notes/commits`、reflog 和 dangling/多父对象，而不是把精力放在命令注入上。
- 复用要点：tree 中的对象 ID 是 20 字节二进制而非 40 字符文本；stash 的 untracked 文件通常位于额外 parent；notes 的 tree 项常以被注释对象的 SHA-1 命名。每发现一个 ref 或 parent 都应记录其类型、父子关系和路径，避免只凭 commit message 猜测内容。
