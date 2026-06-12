# Roundabout

## 题目简述

题目给出一个 32 位 Windows PE，文件带 UPX 壳。脱壳后主函数读取 42 字符输入，用 16 字节循环 XOR key 逐字节变换，再与 42 个 DWORD 期望值比较。

XOR key 是字符串 `this_is_not_flag`，提示它只是密钥而不是 flag。按 `plaintext[i] = expected[i] ^ key[i % 16]` 反推即可。

## 解题过程

### 关键观察

原始文件段名为 `UPX0/UPX1`，可直接：

```bash
upx -d Roundabout.exe -o Roundabout_unpacked.exe
```

脱壳后核心逻辑：

```c
gets_s(input, 0x2c);
if (strlen(input) != 42) error;
for (i = 0; i < 42; i++) {
    if ((input[i] ^ key[i % 16]) != expected[i]) error;
}
```

关键数据：

```text
key = "this_is_not_flag"
expected =
44 10 2e 12 32 0c 08 3d 56 0a 10 67 00 41 00 01
46 5a 44 42 6e 0c 44 72 0c 0d 40 3e 4b 5f 02 01
4c 5e 5b 17 6e 0c 16 68 5b 12
```

### 求解步骤

```python
key = b"this_is_not_flag"
expected = [
    0x44,0x10,0x2e,0x12,0x32,0x0c,0x08,0x3d,
    0x56,0x0a,0x10,0x67,0x00,0x41,0x00,0x01,
    0x46,0x5a,0x44,0x42,0x6e,0x0c,0x44,0x72,
    0x0c,0x0d,0x40,0x3e,0x4b,0x5f,0x02,0x01,
    0x4c,0x5e,0x5b,0x17,0x6e,0x0c,0x16,0x68,
    0x5b,0x12,
]

flag = bytes(v ^ key[i % len(key)] for i, v in enumerate(expected))
print(flag.decode())
```

输出：

```text
0xGame{b8ed8f-af22-11e7-bb4a-3cf862d1ee75}
```

## 方法总结

- 核心技巧：UPX 脱壳后识别循环 XOR 校验并反推明文。
- 识别信号：段名 `UPX0/UPX1`、壳前函数极少、壳后验证逻辑明显时，应先脱壳再分析。
- 复用要点：DWORD 目标数组里只使用低字节时，提取时不要把 4 字节整数误当成连续密文；循环 key 的取模长度必须按字符串真实长度计算。
