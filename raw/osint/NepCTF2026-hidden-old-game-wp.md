# NepCTF2026 HiddenOldGame Writeup

## 题目简述

题目给出一张老式纸牌游戏截图和一个 AES-GCM 解密脚本。脚本不直接询问游戏名，而是要求填写目标可执行文件名，并把文件名与可执行文件完整内容的 BLAKE2b 摘要共同用于派生密钥。必须先通过公开互联网资料识别并找回正确版本的旧游戏，再验证文件散列。

## 解题过程

截图中最有辨识度的线索是：

- 窗口标题出现 `Super80`；
- 牌桌上出现 `15X4=60`；
- 玩法属于 80 分、拖拉机或炒地皮一类纸牌游戏。

据此可将目标定位为“炒地皮 6.2.2（新超级 80 分）Super80”。历史下载包名为：

```text
S80Setup6.2.2.zip
```

旧下载站的跳转地址可能已经失效或返回 403，因此不能只依赖网页按钮。需要结合网页存档、页面中的软件编号和实际下载响应核对版本。下载包内为 Setup Factory 7.0 安装器。为避免在宿主机直接运行未知老软件，可先静态查看压缩包与安装器内容；若工具无法解包，再在隔离虚拟机中安装。

安装器内的目标文件为 `%AppFolder%\Super80.exe`，正确样本信息如下：

```text
文件名：Super80.exe
大小：8237056 bytes
MD5：6497de759f1b0bb1d19355837388e111
SHA1：4c1a69b0bf5ca873bb493e7bf98c7b7aa99d21e8
```

题目脚本对 `FILENAME` 调用了 `.lower()`，所以参与散列的字符串是 `super80.exe`。在区分大小写的文件系统中，需要把文件复制或重命名为同名小写文件。密钥派生和解密过程为：

```python
import hashlib
from Crypto.Cipher import AES

filename = "super80.exe"
data = open(filename, "rb").read()

file_hash = hashlib.blake2b(data).digest()
name_hash = hashlib.blake2b(filename.encode()).digest()
derived = hashlib.blake2b(name_hash + file_hash).digest()

key = derived[:16]
nonce = derived[16:32]

ciphertext = bytes.fromhex(
    "ec9f0a650c4aa727c328aa77ef05312d1df48f67aa3e86570ba107de42ee6630"
    "55e279541acf5375d14f21ce949bab2d3e46401104c7f1e636"
)
tag = bytes.fromhex("bbe86c0017179bd8aa146aff509c5f3a")

cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
flag = cipher.decrypt_and_verify(ciphertext, tag)
print(flag.decode())
```

输出为：

```text
NepCTF{Y0u_Have_FouNd_Mem0ries_IN_the_Internet_Gr4veyard}
```

## 方法总结

本题不是只回答游戏名称，而是要求找到“字节级完全一致”的历史版本。网页标题、压缩包名和安装器信息用于定位，文件大小与 MD5/SHA1 用于最终验真；只有这些信息全部一致，BLAKE2b 派生出的 AES-GCM key 和 nonce 才能通过认证标签。对来源不明的旧程序应优先静态解包，必要时也只在隔离环境运行。
