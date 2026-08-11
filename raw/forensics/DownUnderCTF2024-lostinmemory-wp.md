# Lost in Memory

## 题目简述

题目原始附件为 Windows 7 内存镜像，要求从控制台历史、进程树和 PowerShell 内存字符串恢复四项行为证据：初始脚本、载入模块、两个 DLL 以及新建账户密码。最终将四项小写结果以 `_` 拼接。

## 解题过程

先用与镜像匹配的 Volatility profile 查看控制台历史：

```text
vol.py -f EMU-OUTPOST.raw --profile=Win7SP1x86_23418 consoles
```

PowerShell 控制台显示运行了 `C:\Users\emu\Downloads\monkey.doc.ps1`，因此第一项为 `monkey.doc.ps1`。再查看进程树：

```text
vol.py -f EMU-OUTPOST.raw --profile=Win7SP1x86_23418 pstree
```

可以看到 PID 1136 的 `powershell.exe` 派生出 PID 2520 与 PID 3268。导出 PID 2520 内存并检索命令和下载字符串：

```text
vol.py -f EMU-OUTPOST.raw --profile=Win7SP1x86_23418 memdump -p 2520 --dump-dir .
```

字符串记录表明脚本下载 `reflect.ps1` 后调用 `Invoke-ReflectivePEInjection`，并对 `emu.dll` 与 `kiwi.dll` 执行该模块。按题意字母排序，第三项应写成 `emu.dll-kiwi.dll`。

另一个 PowerShell 字符串把命令拆为单字符/短片段相加。还原拼接后得到：

```powershell
net user admin 5up3r-5ecur3 /add; net localgroup administrators admin /add
```

四项拼接为：

```text
DUCTF{monkey.doc.ps1_invoke-reflectivepeinjection_emu.dll-kiwi.dll_5up3r-5ecur3}
```

## 方法总结

内存取证应把控制台历史、父子进程关系和进程私有字符串交叉验证：前者给出初始落地脚本，进程树确定应导出的 PowerShell 进程，内存字符串再补足下载载荷和混淆命令。对简单 PowerShell 字符串拼接，应先还原文本再判断命令语义，避免把片段误当作独立 IOC。
