# dualzip

## 题目简述

服务提供两个上传接口：`/unzip` 在随机临时目录中执行 `unzip -:`，`/un7z` 执行 `7z x -y`；两者都会把提取目录重新打包返回。`unzip -:` 接受 ZIP 成员中的 `../` 路径且不阻止符号链接，`7z` 又支持分卷 ZIP（`.z01`、`.z02`、`.zip`）。两个解压器的语义差异可以被串成任意文件读取：先将伪造分卷和指向 `/flag` 的链接写到可预测的临时路径，再让 7z 将该链接当作分卷内容读取。

## 解题过程

应用为每次上传的原始字节计算 SHA-256，并把文件写到固定命名路径：

```python
h = sha256(filedata).hexdigest()
zippath = f"/tmp/{h}.zip"
extractdir = f"/tmp/{secrets.token_hex(8)}"

subprocess.run([*extract_cmd, zippath], cwd=extractdir)
subprocess.run([*create_cmd, finalzippath, extractdir])
```

因此只要离线构造的第二阶段 ZIP 字节为 `finalmain`，就能预先算出 `H = SHA256(finalmain)`，从而知道 7z 会寻找的 `/tmp/H.z01` 与 `/tmp/H.z02`。

第一阶段提交给 `/unzip` 的 setup ZIP 包含两个 Unix 风格成员：

```text
../H.z01   普通文件：分卷 0 的 local file header
../H.z02   符号链接：链接目标为 /flag
```

成员名中的 `../` 依靠 `unzip -:` 从随机提取目录越界到 `/tmp`；第二个成员通过 ZIP 中央目录的 Unix 外部属性标记为符号链接。这样 `/tmp/H.z01` 和 `/tmp/H.z02` 已经就位，且 `H.z02` 实际解析到 `/flag`。

第二阶段将 `finalmain` 上传到 `/un7z`。它是同一份多卷 ZIP 的最终 `.zip` 分卷，中央目录声明数据从前两个分卷连续而来。7z 读取 `.z02` 时跟随符号链接，将 `/flag` 的字节当作目标文件（例如 `test.txt`）的数据提取到临时目录；服务随即重新压缩该目录并返回。解开响应 ZIP 即得到：

```text
DUCTF{what_the_z1p_just_happened_38442190}
```

## 方法总结

- 核心技巧：把路径穿越、ZIP 符号链接和分卷归档解析组合成跨解压器的任意文件读取。
- 识别信号：同一服务把上传归档交给多个工具处理，且临时文件名由可预测哈希决定时，应检查一个工具能否为另一个工具预置文件、链接或分卷。
- 复用要点：修复不能只依赖单个解压器的默认保护；应拒绝绝对路径、`..`、符号链接和分卷归档，在隔离目录内解压后以真实路径校验所有输出，并避免把同一可预测临时目录暴露给后续解析器。
