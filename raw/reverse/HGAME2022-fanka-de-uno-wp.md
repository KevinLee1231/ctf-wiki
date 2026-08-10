# 饭卡的uno

## 题目简述

附件是 Arduino 固件的 Intel HEX 文件。虽然官方把题目放在 IoT，但不需要串口、总线、烧录或硬件交互；flag 以明文字符串直接存在固件数据中，决定性步骤只是查看或转换固件内容，因此归入 Reverse。

## 解题过程

Intel HEX 是 ASCII 文本格式，每行记录地址、类型、数据和校验和。直接用十六进制编辑器搜索 `hgame` 即可看到明文。也可以先把 HEX 转成原始二进制，再提取可打印字符串：

```bash
avr-objcopy -I ihex -O binary firmware.hex firmware.bin
strings -a firmware.bin | grep 'hgame{'
```

在反汇编或字符串视图中，对应区域为：

```asm
db 68h
db 'game{F1rst_5tep_0F_IOT}', 0
```

首字节 `0x68` 是字符 `h`，与后续字符串拼接后得到：

```text
hgame{F1rst_5tep_0F_IOT}
```

原 PDF 只说明“HEX 打开即可发现明文”，没有打印具体内容；固件字符串和 flag 通过 [YuGao 的饭卡的uno记录](https://sxyugao.top/p/d379320f) 补全，正文已给出字节与字符串的直接对应关系。

## 方法总结

不要仅凭 Arduino 或固件外壳就把题目判为硬件方向。先做最小静态检查：确认文件格式、转换为二进制并运行字符串扫描。本题没有额外编码或控制流，明文字符串就是完整答案。
