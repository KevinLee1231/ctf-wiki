# Hackergame2020 一闪而过的 Flag WP

## 题目简述

附件 `Untitled01.exe` 启动后会在控制台输出 flag，但双击运行时，新建的控制台窗口会随着进程结束立即关闭，所以只能看到内容一闪而过。

这不是二进制算法逆向或漏洞利用题，核心只是从已有终端运行命令并保留标准输出，故暂归 `_unclassified`。

## 解题过程

不要在资源管理器中双击程序。先打开 PowerShell，切换到附件所在目录，再从这个已经存在的终端启动它：

```powershell
Set-Location "D:/path/to/challenge"
& ".\Untitled01.exe"
```

子进程退出后，PowerShell 窗口仍然存在，输出便留在屏幕上。也可以把标准输出重定向到文件，避免字体造成视觉混淆：

```powershell
& ".\Untitled01.exe" > ".\output.txt"
Get-Content -LiteralPath ".\output.txt"
```

读取到：

```text
flag{Are_you_eyes1ght_g00D?_can_you_dIst1nguish_1iI?}
```

末尾的 `1iI` 分别是数字 `1`、小写 `i`、大写 `I`。Windows 旧版默认控制台字体中 `1` 与 `I` 很接近，截图或录屏辨认容易抄错；直接复制文本最可靠。

## 方法总结

图形界面启动的控制台程序会让窗口生命周期与进程绑定，而在已有 shell 中运行时，终端由 shell 持有。遇到“窗口一闪而过”，优先从 PowerShell 或 CMD 启动并捕获 stdout/stderr，不需要先反编译可执行文件。
