# DownUnderCTF 2023 static file server Writeup

## 题目简述

服务使用 aiohttp 3.8.5，把 `./files` 映射到 `/files`，并设置 `follow_symlinks=True`。Dockerfile 显示 flag 位于静态目录之外的 `/flag.txt`。问题出在静态资源处理器跳过了目录边界检查。

## 解题过程

处理器会把请求路径拼接到静态目录并调用 `resolve()`。正常情况下还会执行：

```python
filepath.relative_to(self._directory)
```

该语句会拒绝解析后逃出静态根目录的路径。然而 aiohttp 3.8.5 的实现只在 `follow_symlinks` 为假时执行它；题目恰好将该选项设置为真，因此 `../` 可以越过 `./files`。

请求工具通常会规范化路径，所以要让 curl 保留原始的 `../`：

```bash
curl --path-as-is "http://<HOST>:<PORT>/files/../../../../../../flag.txt"
```

路径解析到容器根目录的 `/flag.txt`，响应为：

```text
DUCTF{../../../p4th/tr4v3rsal/as/a/s3rv1c3}
```

## 方法总结

漏洞并非普通符号链接本身，而是 `follow_symlinks=True` 分支连带跳过了“解析后路径仍位于静态根目录内”的验证。复现路径穿越时还要注意客户端和中间代理的 URL 规范化，必要时显式保留原始路径。
