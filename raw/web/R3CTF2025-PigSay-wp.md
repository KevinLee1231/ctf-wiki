# PigSay

## 题目简述

题目是 Litestar 编写的 PigSay 加解密站点，支持上传 ZIP、RAR、7z 和 `tar.gz`，解压后处理顶层 `*.txt`。管理员接口会验证 HS256 JWT，再以 `r3ctf` 用户执行 `/app/upgrade.sh`。

完整利用链依赖题目固定的 `python:3.14.0b2-alpine3.21`：

1. 用 tar 软链接读取 `/proc/self/environ`，取得 `JWT_KEY`；
2. 从 Litestar 自动生成的 `/schema` 找到随机管理员路由；
3. 利用 CVE-2025-4517 的 tar 提取路径问题覆盖可写的 `pigsay` 启动器；
4. 伪造管理员 JWT 调用升级接口，让 `uvx pigsay` 执行被覆盖的脚本并返回 flag。

Python 3.14.0b3 已修复相关问题，因此镜像的 beta 小版本是必要条件。

## 解题过程

### 泄露隐藏路由与 JWT_KEY

管理员路由在模块导入时随机生成：

```python
@post(f"/api/admin/upgrade/{uuid4().hex}")
```

但 Litestar 默认公开 OpenAPI schema。访问 `/schema`，在返回内容中搜索：

```text
/api/admin/upgrade/<32位十六进制字符串>
```

即可得到本实例的真实路径。

应用由下面的命令启动：

```sh
JWT_KEY=<随机值> PIG_KEY=<随机值> uv run app.py
```

虽然 `app.py` 用 `environ.pop()` 删除了 Python 字典中的变量，进程最初的环境块仍可从 `/proc/self/environ` 读取。构造 `tar.gz`，依次放入三个符号链接：

```python
add("a/",        type=SYMTYPE, linkname="../")
add("b/",        type=SYMTYPE, linkname="a/../../")
add("data.txt/", type=SYMTYPE, linkname="b/proc/self/environ")
```

上传至 `/api/file/encrypt`。解压后，服务的：

```python
item.read_text() for item in tmp_dir.glob("*.txt")
```

会跟随 `data.txt` 软链接读取环境文件，再把内容用 PigSay 加密放进 JSON。将该密文发给公开的 `/api/decrypt`，即可还原以空字节分隔的环境变量，并提取：

```text
JWT_KEY=<secret>\x00
```

### 用 CVE-2025-4517 覆盖 pigsay 启动器

管理员的 `upgrade.sh` 本身不可写，但最后会运行：

```sh
uvx pigsay encrypt "..."
```

实际执行文件是：

```text
/home/r3ctf/.local/share/uv/tools/pigsay/bin/pigsay
```

这个文件归 `r3ctf` 用户所有，可以被覆盖。官方 PoC 针对 `tarfile.extractall()` 的 CVE-2025-4517 构造一串接近 `PATH_MAX=4096` 的深目录与符号链接，再建立逃逸链接和指向目标的硬链接，最后用同名普通文件写入内容。核心归档顺序是：

```text
16 级超长目录及对应短符号链接
超长路径下指回父目录的符号链接
escape -> 上述链接
data.txt 硬链接 -> escape/../home/r3ctf/.../bin/pigsay
同名 data.txt 普通文件，携带可执行脚本
```

利用载荷可写成：

```sh
#!/bin/sh
echo "flag: $(cat /app/flag_*)"
```

归档中的普通文件权限设为 `0777`。将该 `tar.gz` 再上传到文件处理接口，易受攻击版本会在临时解压目录之外覆盖 `pigsay` 启动器。

### 伪造管理员身份并触发命令

用泄露的 `JWT_KEY` 生成：

```python
jwt.encode(
    {"role": "admin"},
    JWT_KEY,
    algorithm="HS256",
)
```

向 `/api/admin/upgrade/<uuid>` 发送 POST 请求，并设置请求头：

```text
r3-token: <伪造JWT>
```

升级逻辑以 `r3ctf` 用户执行 `upgrade.sh`。当脚本运行 `uvx pigsay` 时，刚写入的 shell 脚本被执行；`/app/flag_*` 虽由 root 创建，但权限为 `0744`，因此仍可读取。命令输出由 `check_output()` 捕获并放入升级接口 JSON 的 `data` 字段，直接得到 flag。

## 方法总结

本题不是单一 CVE 的直接套用，而是四个边界连续失守：OpenAPI 暴露随机路由，压缩包软链接把本地文件变成 Web 回显，进程环境泄露 JWT 密钥，tar 任意写最终劫持升级脚本调用的可写工具入口。

源码还支持 RAR，比赛中确实存在借助 RAR 文件读写的非预期路线；本文保留与题目 Python 版本直接对应的预期 tarfile 解法。完整 CVE 归档构造与服务端交互可见出题人的 [PigSay 出题记录](https://www.woodwhale.cn/2025-r3ctf-misc-pigsay-wp/)，正文已经概括了每个归档成员的作用和最终执行目标。
