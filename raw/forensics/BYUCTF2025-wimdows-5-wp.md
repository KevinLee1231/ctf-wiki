# Wimdows 5

## 题目简述

题目沿用 [Wimdows 虚拟机证据](https://byu.box.com/v/byuctf-wimdows)，要求找到攻击者留下的 SYSTEM 提权后门。Sysmon 注册表事件记录了 `Image File Execution Options\sethc.exe` 下 `Debugger` 值的修改。

这是经典 Sticky Keys/IFEO 后门：Windows 在登录界面以高权限启动 `sethc.exe` 时，IFEO Debugger 会改为启动攻击者指定的程序。

## 解题过程

在 Sysmon Registry SetValue 记录中搜索 `sethc.exe`、`Image File Execution Options` 或 `Debugger`。本题写入的完整命令为：

```powershell
REG ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\sethc.exe" `
  /t REG_SZ /v Debugger `
  /d "C:\windows\system32\cmd.exe #byuctf{00p5_4ll_b4ckd00r5_139874}" /f
```

之后在锁屏界面触发 Sticky Keys，本应启动 `sethc.exe` 的 SYSTEM 流程会由 IFEO 改为 `cmd.exe`。攻击者还把 flag 作为命令行注释放在 Debugger 字符串中，因此无需实际触发后门即可从注册表事件直接提取：

```text
byuctf{00p5_4ll_b4ckd00r5_139874}
```

## 方法总结

- 核心技巧：从 Sysmon 注册表修改中识别 `sethc.exe` 的 IFEO Debugger 劫持，并读取其命令行证据。
- 识别信号：辅助功能可执行文件与 `Image File Execution Options\...\Debugger` 组合出现时，应优先判断为登录界面 SYSTEM 后门。
- 复用要点：取证时既要记录键路径和值，也要解释触发上下文；删除后门时还应检查其他辅助功能程序是否存在同类劫持。
