# bi0sCTF 2024 - Batman Investigation III: Th3 Sw0rd 0f Azr43l

## 题目简述

题目提供 Windows 磁盘取证镜像和交互式答题服务，共 11 问。攻击链从恶意 Office 宏开始，落地 Rust 勒索软件；后续需要从进程转储恢复 ChaCha20 nonce 与文件密钥，解密浏览器、Notepad++、二维码和视频工件，并完成时间线与凭据恢复。

尽管包含勒索软件逆向，最终问题依赖注册表、执行痕迹、内存、浏览器和多媒体文件之间的证据重建，因此归入 forensics。

## 解题过程

### 从磁盘镜像建立感染时间线

在 FTK Imager 中载入 AD1，先导出 SYSTEM、用户目录、Prefetch、事件日志、Office 最近使用记录和浏览器数据。SYSTEM hive 给出主机名与时区：

```text
Q1  DESKTOP-37SCR9U_GMT-Standard-time
```

Office LNK 和最近文件记录指向 `Eula.docm`。其 `AutoOpen`/`Document_Open` 宏从远端下载 `mssetup.exe` 到 `%TEMP%`，再通过 `ShellExecute` 的 `runas` 方式执行。文件哈希与时间线答案为：

```text
Q2  Eula.docm_c814d5d719bea855b4da9db1e1a67f0e
Q3  mssetup.exe_35133696c5d4aca60d65f59138810e4d
Q4  21-02-2024-16-33
```

事件、Prefetch 和文件时间应互相印证。单靠 Office 文档的修改时间无法证明宏实际执行。

### 还原 Rust 勒索软件

`mssetup.exe` 经 UPX 打包，是 Rust 编写的勒索软件。它使用多项反分析条件：`IsDebuggerPresent`、最近输入不超过 60 秒、鼠标坐标变化，以及特定次数的左右键点击。需要动态观察时，应在隔离虚拟机中修补这些分支，不能在宿主机直接运行样本。

样本用 ChaCha20 加密文件。每个文件使用随机 32 字节密钥；程序还生成 24 字节随机字符串，其中前 12 字节作为 nonce，并把 nonce 发送到：

```text
Q5  192.168.1.33:6969
```

密钥没有真正离开本地，而是先与常量逐字节异或，再作为 32 字节尾部附加到 `.azr43l`：

```python
mask = b"98238588864125956469313398338937"
key = bytes(a ^ b for a, b in zip(blob[-32:], mask))
ciphertext = blob[:-32]
```

因此未知量只剩 nonce。从 `mssetup.exe` 进程转储中枚举重复出现的 12 字节字母数字串，用已知 PNG 密文逐个解密并检查 `\x89PNG\r\n\x1a\n` 文件头，可锁定：

```text
nonce = F2E44EB1F60F
Q6    = cec72ce0ac9b9e20288bb66bf5f1c95a
```

Q6 是 nonce 的 MD5。确定 nonce 后，每个文件都从自身末尾恢复 key，再对去掉 32 字节尾部的密文执行 ChaCha20 解密。

### 恢复 Notepad++ 与 Chrome 凭据

解密 Notepad++ 备份后得到受密码保护的 Pastebin 密码和 Chrome secret key：

```text
Pastebin password: PFA6}>C"lbqUUX1M5i;J
Pastebin content : wO0`7A7L9Wp|?Y<^G
```

题目要求分别计算 MD5：

```text
Q7  1af43ebd3fe46a870f2bca51c525af76_b8df2e33554bbbf7e58c793ed2e401a1
```

Chrome 阶段需组合 `Local State` 中的加密主密钥、用户 SID 下的 DPAPI 材料和 `Login Data` SQLite 数据库。解开 Traboda 记录后得到：

```text
Q8  jeanpaulvalley33@gmail.com_3mZxVDpGUhnm2LL
```

### 修复二维码与视频

解密后的 `15.png` 是结构被破坏的二维码。根据三个定位图案、时序线与相邻正常二维码补回缺失模块，扫描结果为：

```text
QR-Code:k4YSM|5^#?34SB$q
Q9  73327e080755e5f7e5ae18a1a4338268
```

MP4 的主要问题是 atom/box 长度字段不正确。按实际文件范围修正大小后播放，在约 1 分 40 秒处取得隐藏文本；仓库校验的是该文本的 MD5：

```text
Q10 d774c3a1cbe3d146c20e4142a8361609
```

AVI 的帧顺序被打乱。每秒约 30 个 `00dc` 视频块，而 I 帧通常明显大于依赖前帧的 P 帧。可以每组取最大块重建仅含 I 帧的视频，也可以极慢速逐帧检查；约 2 分 33 秒处出现目标文本：

```text
Q11 1c7cfa806f06fb78687d26814260b874
```

全部提交后得到：

```text
bi0sCTF{Th3_St_Dum4s_can't_contr0l_th1s_Arch4ng3l_0f_V3ng3nc3_Anym0r3_Rahhh_b63f816}
```

原始宏、反汇编和媒体修复截图可在[官方题解](https://blog.bi0s.in/2024/03/19/Forensics/BatmanInvestigationIII-Th3Sw0rd0fAzr43l-bi0sCTF2024/)中核对；正文已把截图中的关键字符串、算法与提交值全部转为文本。

## 方法总结

本题的主线是“时间线定位样本—逆向确定文件格式—内存补回缺失 nonce—批量恢复各类应用工件—按各格式修复视觉证据”。ChaCha20 密钥就在每个文件尾部，只是与固定 32 字节常量异或；真正需要从进程内存筛选的是 12 字节 nonce。解密后还不能直接结束：Chrome 需要 DPAPI 密钥链，二维码需要结构修补，MP4/AVI 则分别要修正容器长度和处理关键帧顺序。
