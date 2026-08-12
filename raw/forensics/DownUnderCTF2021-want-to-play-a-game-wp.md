# DownUnderCTF 2021 - Want to play a game?

## 题目简述

题目提供一份 Windows 7 x64 内存镜像，要求恢复三项事件事实：感染的恶意软件家族、持久化程序名，以及恶意程序最初运行的目录名。flag 格式为 `DUCTF{malwareName_persistenceName_originatingFolderName}`。主线是从进程、注册表和文件句柄三类内存工件交叉验证，而不是逆向恶意程序本体。

## 解题过程

先用 Volatility 2 判断镜像配置并列出进程：

```bash
vol.py -f JacobsPC.raw imageinfo
vol.py -f JacobsPC.raw --profile=Win7SP1x64 pslist
```

进程列表中 `drpbx.exe`（PID 3628）明显模仿 Dropbox 名称。继续查看加载模块和可执行路径：

```bash
vol.py -f JacobsPC.raw --profile=Win7SP1x64 dlllist -p 3628
```

路径位于：

```text
C:\Users\Jacob\AppData\Local\Drpbx\drpbx.exe
```

`drpbx.exe` 是 Jigsaw 勒索软件的已知进程名；该家族还常把副本写为 `%APPDATA%\Frfx\firefox.exe` 并注册开机启动。检查用户注册表的 Run 键可在本镜像中确认，而不是仅凭文件名猜测：

```bash
vol.py -f JacobsPC.raw --profile=Win7SP1x64 \
  printkey -K 'Software\Microsoft\Windows\CurrentVersion\Run'
```

输出中的持久化命令指向 `Frfx\firefox.exe`，所以第二项为 `firefox.exe`。

最后检查恶意进程持有的文件句柄，寻找与安装目录不同、且可能代表初始投递位置的路径：

```bash
vol.py -f JacobsPC.raw --profile=Win7SP1x64 \
  handles -p 3628 -t File
```

异常句柄指向：

```text
\Device\HarddiskVolume2\Users\Public\Videos\Sample Videos\PJxhJQ9yUDoBF1188y
```

因此三项答案依次是 `jigsaw`、`firefox.exe`、`PJxhJQ9yUDoBF1188y`，组合为：

```text
DUCTF{jigsaw_firefox.exe_PJxhJQ9yUDoBF1188y}
```

## 方法总结

内存事件重建需要把“可疑进程名”“家族情报”“本机持久化证据”和“进程实际句柄”分开验证。进程名只能形成假设；Run 键确认持久化，文件句柄补足投递位置。遇到恶意软件识别题时，优先构造这类多工件证据链，避免只凭搜索结果或单个字符串下结论。
