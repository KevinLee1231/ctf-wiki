# Invaders

## 题目简述

附件 `invaders.exe` 表面上是一款体积异常的太空侵略者游戏。程序由 Godot 引擎打包，真正的校验逻辑并不在游戏脚本本身，而是由脚本解密并落盘的第二个 Windows PE。需要依次完成 Godot 资源提取、GDScript 反编译和原生程序逆向。

## 解题过程

### 从 Godot 程序中提取载荷

运行附件可以看到普通的射击游戏。结合字符串和文件结构可判断它由 Godot 构建，因此先用 GDRE Tools 提取打包资源：

```powershell
.\gdre_tools.exe --headless --extract=.\invaders.exe
```

其中 `game/game.gdc` 是编译后的 GDScript 字节码。题目使用 Godot 4.3，可按对应字节码版本反编译：

```powershell
.\gdre_tools.exe --headless --bytecode="4.3.0" --decompile=.\invaders\game\game.gdc
```

反编译结果表明，游戏启动时会解密一段数据并写入 `gamedata`。用 Detect It Easy 或 PE 头 `MZ` 检查该文件，可确认它才是后续需要分析的 Windows 可执行程序。

### 分析 `gamedata`

`gamedata` 的 TLS 回调会在主函数之前调用 `IsDebuggerPresent`。一旦检测到调试器，程序便长时间休眠。因此动态分析时需要绕过该分支，或使用带 ScyllaHide 的 x64dbg。

主函数创建两个事件和两个工作线程，随后等待两个事件都被置位。线程逻辑分别为：

1. 第一条线程计算输入的 SHA-256，但其控制流最终总会通过，不能真正阻止程序继续。
2. 第二条线程动态解析 `GetEnvironmentVariable`，读取名为 `N0PS_ENV` 的环境变量，并将其与字符串 `S4V3D_3SPE4X` 比较。只有比较成功时才会置位对应事件。

这也解释了直接运行时程序为何一直停留在等待状态：缺失的并不是交互输入，而是进程环境。

### 设置环境变量

在同一个 PowerShell 会话中设置变量并启动载荷：

```powershell
$env:N0PS_ENV = "S4V3D_3SPE4X"
.\gamedata.exe
```

两个事件均满足后，程序输出：

```text
N0PS{Y0u_H4v3_S4V3D_3SPE4X}
```

## 方法总结

本题用 Godot 游戏作为第一层封装，又在解出的 PE 中加入 TLS 反调试和线程同步来分散注意力。有效的分析顺序是先识别打包引擎并恢复 `game.gdc`，再把脚本生成的 `gamedata` 当作独立 PE 分析。最终决定程序能否继续的条件是环境变量 `N0PS_ENV=S4V3D_3SPE4X`，而不是表面的游戏流程或恒成功的 SHA-256 校验。
