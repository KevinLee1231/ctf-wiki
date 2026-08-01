# Not Sure I'll Recover From This

## 题目简述

题目给出 Windows 虚拟机磁盘，要求恢复本地账户的三个安全问题答案。官方简述把 “SAM hive” 与 `C:\Users` 混在了一起；准确做法是从离线 Windows 安装中定位注册表配置单元，再用能解码 Windows 10 安全问题的取证工具读取。

## 解题过程

挂载磁盘或通过取证套件浏览文件系统，定位：

```text
Windows\System32\config\SAM
```

将离线配置单元交给 [NirSoft SecurityQuestionsView](https://www.nirsoft.net/utils/security_questions_view.html)。该工具会从 SAM 配置单元解析本地账户的安全问题与答案；选择 “Load security questions from external drive”，目录直接指向离线系统的 `Windows\System32\config`。

三项记录依次为：

```text
What was your first pet's name?             Jimothy
Where did your parents meet?                Idaho Falls
What's the first name of your oldest cousin? Zephanias
```

按题面顺序转小写并用下划线连接：

```text
byuctf{jimothy_idaho_falls_zephanias}
```

## 方法总结

本题考查的是离线 Windows 账户恢复数据，而不是在用户目录里全文搜索答案。应保留问题顺序、空格规范化规则和 SAM 来源；若用专用 GUI 工具，也要在 WP 中说明它读取了什么证据，避免把“点一下得到答案”当成完整过程。
