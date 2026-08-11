# Akira之瞳-2

## 题目简述

附件 `secret_work.raw` 是 Windows 7 SP1 x64 内存镜像。题目链条依次为：从记事本命令行定位并导出 `dumpme.txt`，取得 `secret.7z` 密码和 LastPass 提示；从内存中的 LastPass 数据恢复 Windows 登录密码，用该密码解开 DPAPI masterkey；再解密旧版 Chrome Cookies，取出 VeraCrypt 容器密码，最后从挂载卷内文件的 NTFS 备用数据流（ADS）中恢复 flag。原 PDF 将其放在 `Misc`，但决定性工作是内存与文件系统证据恢复，因此归入 `forensics`。

## 解题过程

镜像 profile 仍为 `Win7SP1x64`。列出进程和命令行：

```bash
volatility -f secret_work.raw --profile=Win7SP1x64 pslist
volatility -f secret_work.raw --profile=Win7SP1x64 cmdline
```

`notepad.exe` 的 PID 为 456，命令行显示它打开了：

```text
C:\Users\Genga03\Desktop\dumpme.txt
```

文本文件较小，直接从文件对象导出即可：

```bash
volatility -f secret_work.raw --profile=Win7SP1x64 filescan | grep dumpme.txt
volatility -f secret_work.raw --profile=Win7SP1x64 \
  dumpfiles -Q 0x000000007ef94820 -D ./
```

恢复的内容为：

```text
zip password is: 5trqES&P43#y&1TO
And you may need LastPass
```

用该密码解开 `secret.7z`，得到 Chrome 的 `Cookies` SQLite 数据库、用户 DPAPI `Protect` 目录和 VeraCrypt `container`。检查 `Cookies` 表的 `encrypted_value`：其开头不是 Chrome 80 以后常见的 `v10` 或 `v11`，说明它使用旧式 Windows DPAPI，必须取得用户 masterkey。

题目已经给出 LastPass 提示。将 [kevthehermit/volatility_plugins 的 LastPass 插件](https://github.com/kevthehermit/volatility_plugins/tree/master/lastpass) 放入插件目录后运行：

```bash
volatility --plugins=./ -f secret_work.raw \
  --profile=Win7SP1x64 lastpass
```

插件从浏览器进程内存恢复出保存的 Windows 登录凭据：

```text
UserName: windows login & microsoft
Password: vIg*q3x6GFa5aFBA
```

用户 SID 为 `S-1-5-21-262715442-3761430816-2198621988-1001`，`Protect` 目录中目标 masterkey GUID 为 `57935170-beab-4565-ba79-2b09570b95a6`。在 mimikatz 中执行：

```text
dpapi::masterkey /in:"S-1-5-21-262715442-3761430816-2198621988-1001\57935170-beab-4565-ba79-2b09570b95a6" /sid:S-1-5-21-262715442-3761430816-2198621988-1001 /password:vIg*q3x6GFa5aFBA
```

得到解密后的 masterkey：

```text
3cafd3d8e6a67edf67e6fa0ca0464a031949182b3e68d72ce9c08e22d7a720b5d2a768417291a28fb79c6def7d068f84955e774e87e37c6b0b669e05fb7eb6f8
```

在同一 mimikatz 会话中用它解密 Cookies：

```text
dpapi::chrome /in:"Cookies"
```

关键记录是名为 `VeraCrypt` 的 Cookie：

```text
!bWjAqM2z!iSoJsV&IRV@AVI1VrtAb
```

用该值作为密码挂载 `container`，卷内得到 `ADS.jpg`。文件名提示 NTFS Alternate Data Streams；列出并导出非默认数据流即可。PowerShell 可使用：

```powershell
Get-Item -LiteralPath ".\ADS.jpg" -Stream *
Get-Content -LiteralPath ".\ADS.jpg" -Stream "<发现的流名>"
```

也可以用 `dir /r`、Sysinternals Streams 或取证软件枚举并导出 ADS。隐藏流中记录的 flag 为：

```text
hgame{Which_0nly_cryin9_3yes_c4n_de5cribe}
```

所有原 PDF 截图都只是终端、数据库字段或纯文本结果，已经转写为可复制内容，没有保留图片。

## 方法总结

本题不是彼此独立的工具堆砌，而是一条凭据解封链：内存中的文本给出压缩包密码，LastPass 内存记录给出 Windows 密码，Windows 密码解开 DPAPI masterkey，masterkey 解密 Chrome Cookie，Cookie 再挂载 VeraCrypt 容器，最终由 NTFS ADS 交付 flag。Chrome `encrypted_value` 的版本前缀决定解密路径；mimikatz 的 masterkey 与 Cookie 解密应保持同一上下文，SID、GUID 和用户密码三者也必须对应。
