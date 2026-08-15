# TakeOver

## 题目简述

题目提供一份 Windows 系统镜像，要求还原一次文件篡改事件的五个字段：原文件名、新文件名、攻击者设置的最后修改时间、修改工具和文件内容。flag 按这些字段依次拼接。

关键证据不是单个文件当前的元数据，而是 PowerShell 历史记录与磁盘中目标文件之间的交叉印证。

## 解题过程

用 Autopsy 导入镜像后，可在“File Views → Documents → Plain Text”中定位 PowerShell 的 ConsoleHost_history.txt。历史记录完整保留了攻击者的操作顺序：

~~~powershell
mkdir h4ck3ed
(Get-Item .\h4ck3ed).Attributes += 'Hidden'
notepad.exe .\My_Diary.txt
mv '.\My_Diary.txt' 'C:\Users\pc\AppData\Local\h4ck3ed\N548der5gg.lol'
Get-ChildItem .\N548der5gg.lol |
    ForEach-Object { $_.LastWriteTime = '12/12/2012 23:59:59' }
cat .\N548der5gg.lol
~~~

由此可以直接确定：

- 原文件名：My_Diary.txt。
- 新文件名：N548der5gg.lol。
- 修改工具：notepad.exe。
- 攻击者主动设置的时间：12-12-2012-23:59:59。

随后沿镜像中的 Users/PC/AppData/Local/h4ck3ed 路径找到 N548der5gg.lol。文件内容为：

~~~text
C4ugh7_m3_r3d_h4nd3d
~~~

Autopsy 界面可能按镜像时区换算显示另一个表面时间；flag 采用历史命令中明确设置的本地值，这也解释了为什么不能只抄文件列表中的显示时间。按题面格式拼接得到：

~~~text
shellmates{My_Diary.txt_N548der5gg.lol_12-12-2012-23:59:59_notepad.exe_C4ugh7_m3_r3d_h4nd3d}
~~~

官方 WP 的四张 Autopsy 截图只承载上述目录、命令和文本信息，均已转写，因此不再保留截图。

## 方法总结

Windows 文件篡改调查应把命令历史、目录位置、文件内容和时间戳放进同一证据链。单独相信最后写入时间容易被 timestomping 欺骗；命令历史既能解释重命名和隐藏目录，也能指出时间是人为设置的。界面截图中的时区显示与命令原值冲突时，应先明确题目要求的是哪个语义，再选择用于 flag 的时间。
