# ezUPX

## 题目简述

附件是一个 64 位 Windows PE，查壳结果显示入口位于 `UPX1`，并具有 UPX 3.91 的特征。脱壳后程序只对一组固定字节逐位异或 `0x32`，解码结果就是 flag。

## 解题过程

Exeinfo PE 等识别工具会报告 UPX 签名。使用 [UPX 官方工具](https://github.com/upx/upx) 的 `-d` 参数恢复原始程序：

```powershell
upx.exe -d ".\ezUPX.exe"
```

脱壳后在反编译结果中可以看到 37 字节数组和固定异或循环。无需保留完整程序，下面的 Python 代码已经包含复现解码所需的全部数据：

```python
ciphertext = bytes(
    [
        0x64, 0x7B, 0x76, 0x73, 0x60, 0x49, 0x65, 0x5D, 0x45, 0x13,
        0x6B, 0x02, 0x47, 0x6D, 0x59, 0x5C, 0x02, 0x45, 0x6D, 0x06,
        0x6D, 0x5E, 0x03, 0x46, 0x46, 0x5E, 0x01, 0x6D, 0x02, 0x54,
        0x6D, 0x67, 0x62, 0x6A, 0x13, 0x4F, 0x32,
    ]
)

plaintext = bytes(value ^ 0x32 for value in ciphertext).rstrip(b"\x00")
print(plaintext.decode())
```

运行后得到：

```text
VIDAR{Wow!Y0u_kn0w_4_l1ttl3_0f_UPX!}
```

## 方法总结

- 核心技巧：识别并脱去标准 UPX 壳，再还原固定 XOR 循环。
- 识别信号：PE 节名为 `UPX0`/`UPX1`，入口字节和查壳工具均指向 UPX。
- 复用要点：标准 UPX 优先使用官方 `upx -d`；脱壳只是恢复分析条件，真正的 flag 仍要从内部校验或解码逻辑中取得。
