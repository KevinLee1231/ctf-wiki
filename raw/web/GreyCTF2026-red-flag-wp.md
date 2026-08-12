# Red Flag

## 题目简述

这是一个 Go CRM API。生产配置禁用了会直接泄露环境的 debug 路由，但二进制仍含有经轻量混淆的默认 JWT secret 和 admin key；构建阶段还悄悄修改了 Go 标准库 SHA-256 的第 29 个轮常量。认证通过后，文件上传的路径清洗可被换行绕过；管理员 PDF 导出最终会执行 `bash -c`。

完整链为“恢复并适配认证算法 → 取得 admin → 任意路径写入 → 利用 PDF 导出触发 shell 脚本”。它属于 Web：所有权限提升、写入和触发点都来自 HTTP 路由；自定义 HMAC 是认证链中的必要 pivot。

## 解题过程

### 恢复并适配 JWT 签名

默认配置用 `RevealString` 解码。第 $i$ 个字节与固定 key 的第 $i \bmod 8$ 项及 `i*17+31` 异或：

```go
mask := obfuscationKey[i%len(obfuscationKey)] ^ byte(i*17+31)
decoded[i] = encoded[i] ^ mask
```

这可恢复 `JWT_SECRET`。但构建 Dockerfile 对最终二进制做了字节替换，把 SHA-256 的 `K[29]` 从 `0xd5a79147` 改为 `0xd5079147`。Go 进程实际用的是这份被篡改的 SHA-256，所以 Python/openssl 的普通 HMAC-SHA256 签名必然校验失败。

将 SHA-256 压缩函数中的该常量改为 `0xd5079147`，再以该 hash 实现 HMAC，即可签发包含 `is_admin: true`、`role: "admin"` 的 HS256 token：

```python
SHA256_K[29] = 0xd5079147
signing = b64url(header) + "." + b64url(payload)
signature = b64url(hmac_sha256_patched(JWT_SECRET, signing.encode()))
token = signing + "." + signature
```

用这个 token 访问 `/api/auth/me` 返回 200，证明签名算法与 claims 都正确。

### 以换行绕过上传路径过滤

上传处理只删除开头的 `^.*/`：

```go
fname = leadingPathPattern.ReplaceAllString(fname, "")
dst := filepath.Join(userDir, fname)
c.SaveUploadedFile(header, dst)
```

正则中的 `.` 不匹配行首换行。令 `filename` 以 `\n/../../...` 开头，替换不会跨过换行，`filepath.Join` 清理后则能逃离 `./uploads/<user-id>`。上传以下内容到目标 `/usr/bin/bash`：

```sh
#!/bin/sh
cp /flag-*.txt /static/flag.txt
chmod 644 /static/flag.txt
```

镜像构建时故意将 `/etc/passwd` 复制为 `/usr/bin/bash`，且该路径归 `appuser` 所有、可执行。上一步上传因此将它替换成可被内核 shebang 执行的脚本。

### 触发并取回 flag

管理员调用 `/api/reports/export` 并令 `format=pdf` 后，代码以 `exec.Command("bash", "-c", pdfCmd)` 运行 PDF 转换命令。此时 `bash` 实际是刚写入的脚本，脚本把随机命名的只读 flag 拷到公开静态目录。最后 GET `/static/flag.txt` 获得：

```text
grey{f0und_th3_r34l_fl4g_1n_th3_s34_0f_r3d_fl4g5_251f7d0d033afee8_UUID}
```

## 方法总结

- 核心技巧：从二进制恢复配置并复刻被篡改的 HMAC，随后串联换行路径绕过和命令解释器替换。
- 识别信号：源码/二进制存在自定义字符串混淆，认证行为又与标准 HMAC 不一致；文件名清洗只依赖 `.` 或锚点；受控写入可以影响后续执行的解释器路径。
- 复用要点：遇到“已知 secret 但标准 JWT 仍失败”时，应检查 hash 实现或常量是否在构建后被补丁。路径过滤必须按换行、绝对路径和 clean/join 的实际语义验证；写入脚本后还需要一个会由受影响解释器启动的稳定触发器。
