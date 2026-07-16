# 壳艺大师

## 题目简述

原程序被 UPX 打包，直接载入 IDA 时控制流和函数识别异常。脱壳后可见校验逻辑：用户输入与一段内置 44 字节数据逐字节比较，每个输入字节先与循环密钥 `The0xGameKey` 异或。

## 解题过程

先检查节区名和打包器特征：样本存在典型的 `UPX1` 节区，查壳结果指向 x64 UPX 3.9—4.0，可直接尝试官方解包功能。

使用官方 UPX 解包，再分析输出文件：

```bash
upx -d challenge.exe -o unpacked.exe
```

脱壳后的 `main` 中可见循环异或和逐字节比较；密钥由 `sub_1400016E0` 初始化为 `The0xGameKey`。关键逻辑可简化为：

```c
for (int i = 0; i < 44; i++) {
    if ((input[i] ^ key[i % strlen(key)]) != ciphertext[i]) {
        return failure;
    }
}
```

内置密文在栈上以 11 个 32 位立即数写入。x86-64 使用小端序，必须先把每个 DWORD 按低字节在前还原，再与循环密钥异或。下面的数组就是反汇编中出现的 11 个立即数：

```python
from struct import pack

dwords = [
    0x51221064, 0x0F1A2215, 0x18017C06, 0x1D560A6C,
    0x08577E4B, 0x4C512848, 0x53074560, 0x5E4C771E,
    0x4F537B5D, 0x52075961, 0x1007741C,
]
ciphertext = pack("<11I", *dwords)
key = b"The0xGameKey"
flag = bytes(c ^ key[i % len(key)] for i, c in enumerate(ciphertext))
print(flag.decode())
```

脚本恢复出：

```text
0xGame{bc7da8b3-396e-c454-bcf0-3806651bbd3f}
```

## 方法总结

查壳后应优先使用对应打包器的正规解包功能，失败时才考虑动态脱壳。提取内置整数数组时要明确元素宽度和字节序；循环异或的逆运算仍是同一个异或操作，按正确顺序恢复密文字节即可。
