# UMDCTF 2019 - Flash Forensics I

## 题目简述

附件名为 `ubuntu_live.img`，表面上是一段 Ubuntu 安装介质。题目提示 U 盘中曾保存 flag，需要从镜像的原始字节中定位残留数据。

## 解题过程

先做低成本检查，不急于挂载文件系统：

```bash
file ubuntu_live.img
strings -a -t d ubuntu_live.img | grep 'UMDCTF'
```

`strings` 在十进制偏移 `13734460` 附近直接找到：

```text
UMDCTF-{n0_str1ngs_att4ch3d_h3re}
```

这说明 flag 作为连续可打印字符串残留在镜像未被文件系统索引直接展示的区域，不需要恢复完整 Ubuntu 目录树。

仓库 `README.md` 的 Flag 字段有一个格式问题：它包含 65 个十六进制字符，末尾多出一个 `1`。对恢复出的 flag 计算 SHA-256，结果是：

```text
7c9f80233080f78deab6055878812375ec932d1098a7b40af648fd626e19bb60
```

该值与 README 的前 64 位一致。

## 方法总结

镜像取证应先从文件类型、字符串、分区表和文件系统结构这些低成本证据开始。即便文件未出现在目录中，原始扇区仍可能保留明文。校验摘要本身也可能录入错误，因此要保留完整计算过程，并明确区分“恢复结果”和“元数据笔误”。
