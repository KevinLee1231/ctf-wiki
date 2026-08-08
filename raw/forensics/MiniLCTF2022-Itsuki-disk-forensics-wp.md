# MiniLCTF2022 含樹イツキの憂鬱 Writeup

## 题目简述

附件是一份 Windows 磁盘镜像。调查目标不是单点文件恢复，而是串联勒索程序、已加密文档、iPhone 备份、VeraCrypt 容器和系统缓存中的多层证据。镜像内的可执行文件具有恶意行为，只应静态分析或在隔离环境中检查，不能在宿主机直接运行。

## 解题过程

首先检查桌面与下载目录。桌面的 `README.txt` 是勒索提示，`Downloads` 中的“DBT 下载器”会向桌面释放文本，并异或加密用户 `Documents` 下的文件。程序联网时从 `/get_p` 获取在线密钥，失败时才使用默认密钥；`hahaha.log` 证明本次使用的是在线密钥，因此不能直接套默认值。

密钥长度为 16 字节。选取格式已知的 `.docx` 或 `.png`，将密文开头与 ZIP/PNG 标准文件头异或，即可恢复循环异或密钥。多种文件头得到相同结果：

```text
Base64: mkLPGV3n0DiR5yC+XqN5sg==
Length: 16 bytes
```

据此递归恢复以 `.hahaha` 结尾的文件：

```python
from base64 import b64decode
from pathlib import Path

key = b64decode("mkLPGV3n0DiR5yC+XqN5sg==")
for source in Path("Documents").rglob("*.hahaha"):
    data = source.read_bytes()
    plain = bytes(value ^ key[i % len(key)] for i, value in enumerate(data))
    source.with_suffix("").write_bytes(plain)
```

恢复后的 `Documents/好康的/IMG_20220423_114514` 虽伪装成约 50 MB 的 PNG，内容却不符合图像格式；结合系统安装记录可判断它是 VeraCrypt 容器。接着在 `%APPDATA%/Apple Computer/MobileSync/` 找到 iOS 10 的 iTunes 备份，从备忘录数据库 `NoteStore.sqlite` 解码出：

```text
Password: dbtyeyyd5
```

用该密码挂载容器即可取得 flag。另有一条独立佐证：`%SYSTEMROOT%/Temp/` 的快捷方式缓存中存在指向 `z:/flag.txt` 的 `.lnk`，其中 `Z:` 正对应挂载后的容器盘符。

## 方法总结

这道题的证据链是“日志确认密钥来源 → 已知文件头恢复循环异或密钥 → 文件类型复核识别容器 → 手机备忘录恢复口令 → 挂载容器”。取证时要保留每一步的来源和交叉验证，不能只凭扩展名判断文件类型，也不能因程序中存在默认密钥就忽略运行日志。快捷方式指向的盘符不是主解法，却能验证 VeraCrypt 推断与最终文件位置一致。
