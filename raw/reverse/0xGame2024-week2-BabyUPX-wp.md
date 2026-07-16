# BabyUPX

## 题目简述

附件 PE 被 UPX 压缩，原程序入口和校验逻辑在静态分析时不可直接见。文件中可直接找到 `UPX0`、`UPX1` 和 `UPX!` 标记；使用 UPX 自带解压功能脱壳后，`encode` 函数只是交换每个字节的高、低 4 位。

## 解题过程

Detect It Easy 可以识别出 UPX 壳。直接用 IDA 打开加壳文件时，会出现打包文件警告，入口函数也主要是解压 stub，而不是题目主逻辑。

使用 UPX 命令行工具解压到新文件：

```bash
upx -d -o BabyUPX-unpacked.exe BabyUPX.exe
```

重新载入脱壳文件后，`encode` 对每个字节 $x$ 执行：

$$
E(x)=((x\mathbin{\&}0x0f)\ll4)\mathbin{|}(x\gg4)
$$

这个操作是自身的逆运算。程序中的 88 个十六进制字符为：

```text
03877416d656b76383466666435383d223935653d243363603d216933626d2937313665636333383562366d7
```

先按十六进制转成字节，再对每个字节交换半字节：

```python
ciphertext = bytes.fromhex(
    "03877416d656b76383466666435383d223935653d243363603d216933626d2937313665636333383562366d7"
)

plaintext = bytes(
    ((value & 0x0F) << 4) | (value >> 4)
    for value in ciphertext
)
print(plaintext.decode())
```

输出：

```text
0xGame{68dff458-29e5-4cc0-a9cb-971fec338e2f}
```

## 方法总结

UPX 是压缩壳，不等于题目本身的算法。先用文件标记或检测工具确认壳，再用 `upx -d` 恢复原程序；脱壳后的真正变换只是逐字节交换高低半字节，无需依赖在线解码网站。
