# Pickle Rick

## 题目简述

题目由两段 WAV 音频构成。两者都使用 `steghide` 嵌入文本：第一段无密码，提取结果给出第二段的密码；第二段才包含 flag。

## 解题过程

先检查第一段：

```bash
steghide info together-forever-encoded.wav
steghide extract -sf together-forever-encoded.wav
```

直接回车使用空密码即可提取文本：

```text
The password is "big_chungus"!
```

再把该字符串作为第二段音频的口令：

```bash
steghide extract -sf rickroll.wav -p big_chungus
```

第二次提取出的文本为：

```text
UMDCTF-{n3v3r_g0nna_l3t_y0u_d0wn}
```

## 方法总结

多附件隐写题常把一个文件作为另一个文件的密钥来源。应先对每个载体运行 `steghide info`，记录嵌入数据、是否加密和口令要求。本题的顺序由“第一段可空密码提取、其明文明确给出口令”确定，不需要爆破第二段。
