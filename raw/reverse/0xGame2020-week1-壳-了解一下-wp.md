# week1壳，了解一下

## 题目简述

附件是 32 位 Windows PE，并被 UPX 3.96 压缩。`file` 能直接识别 `UPX compressed`，PE 节名中也能看到 `UPX0`、`UPX1`。加壳后的字符串表被压缩，先用 UPX 官方工具还原原始 PE，再做普通静态分析。

## 解题过程

先测试文件是否是有效 UPX 包：

```bash
upx -t 签到2.exe
```

保留原附件，指定新的输出文件进行脱壳：

```bash
upx -d -o 签到2-unpacked.exe 签到2.exe
```

脱壳后运行 `strings -a 签到2-unpacked.exe`，可以看到原程序的完整提示和 flag：

```text
welcome to 0xgame
this problem may be difficult than qiandao
please input a number:
0xgame{up3_is_a_basic_ke}
try again?
```

也可以把脱壳后的文件载入 IDA，在 Strings 窗口中确认该字符串及其引用。这里的关键不是破解输入判断，而是先恢复被壳隐藏的静态数据。

## 方法总结

- 核心技巧：识别 UPX 特征并使用 `upx -d` 自动脱壳。
- 识别信号：`file` 报告 UPX、节名为 `UPX0/UPX1`，或文件中存在 `UPX!` 标记。
- 复用要点：脱壳时使用 `-o` 写入新文件，保留原样本；若 `upx -d` 失败，再考虑魔改 UPX 头或动态脱壳。
