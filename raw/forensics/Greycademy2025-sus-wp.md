# Sus

## 题目简述

题目原始证据是一份 Windows 内存转储，SHA-256 为 `b87c02f2e1e854af2e5b01534eda9279a277cb89bcc88069b1d8cc2f1b8644b0`。需要从内存中定位并导出可疑的 `code.exe`，再逆向其固定异或解码逻辑。

## 解题过程

可先用 Volatility 3 查看进程和文件对象，再按实际输出中的 PID 或 `FILE_OBJECT` 地址导出文件：

```bash
python3 vol.py -f memory.dmp windows.pslist
python3 vol.py -f memory.dmp windows.filescan | rg -i 'code\.exe'
python3 vol.py -f memory.dmp windows.dumpfiles --virtaddr <FILE_OBJECT>
```

当前公开仓库没有收录约 4 GiB 的原始转储，只保留官方导出的 `code.exe`、对应 `code.c` 和简要 solve，因此无法在本地诚实复报具体 PID 与文件对象地址；上述值应从各自转储的插件输出取得。仓库内提取物经识别为 64 位 Windows PE，可以继续核对第二阶段。

程序把密文拆成 `enc_a` 与 `enc_b` 两段，逐字节与常量 `0x5a` 异或，然后用解码结果比较输入。最小恢复代码如下：

```python
enc_a = [
    0x3d, 0x28, 0x3f, 0x23, 0x21, 0x28, 0x3f, 0x2c, 0x05, 0x3b,
    0x34, 0x3e, 0x05, 0x3c, 0x35, 0x28, 0x3f, 0x34, 0x29, 0x05,
]
enc_b = [
    0x3d, 0x35, 0x05, 0x32, 0x3b, 0x34, 0x3e, 0x05, 0x33, 0x34,
    0x05, 0x32, 0x3b, 0x34, 0x3e, 0x7b, 0x66, 0x69, 0x27,
]

print(bytes(value ^ 0x5a for value in enc_a + enc_b).decode())
```

本地运行输出：

```text
grey{rev_and_forens_go_hand_in_hand!<3}
```

## 方法总结

这是一条“内存取证 → 可执行文件恢复 → 简单逆向”的混合链。分类仍以从既有内存证据中恢复关键工件为主，归 Forensics；同时应明确原始大文件缺席带来的验证边界，不能捏造 PID。第二阶段的异或解码已由仓库中的官方提取物独立复现，最终 flag 为 `grey{rev_and_forens_go_hand_in_hand!<3}`。
