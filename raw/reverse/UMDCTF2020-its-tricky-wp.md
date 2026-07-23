# It's Tricky

## 题目简述

附件是 Windows DLL。导出函数名故意写成 `D11Main`，其中是数字 `1`，并要求通过 `rundll32.exe` 调用。DLL 初始化一组字节，导出函数在参数校验通过后逐字节减 10 并输出。

## 解题过程

在 PE 导出表中找到 `D11Main`。进程附加时初始化的 15 字节数组为：

```text
5f 57 4e 4d 5e 50 37 85 5c 7f 78 4d 57 4e 87
```

每个字节减去十进制 10：

```python
encoded = bytes([
    0x5f, 0x57, 0x4e, 0x4d, 0x5e,
    0x50, 0x37, 0x85, 0x5c, 0x7f,
    0x78, 0x4d, 0x57, 0x4e, 0x87,
])
print(bytes((value - 10) & 0xff for value in encoded).decode())
```

得到：

```text
UMDCTF-{RunCMD}
```

程序还会把 `rundll32` 传入的第四个参数与循环 XOR 后的目标比较。动态执行时使用的形式是：

```text
rundll32.exe ItsTricky.dll,D11Main <argument>
```

但静态恢复 flag 不依赖满足该命令参数检查。

## 方法总结

DLL 逆向要同时检查入口点初始化和导出函数，单看 `DllMain` 容易漏掉真实逻辑。还要区分字母 `l`、数字 `1` 等故意混淆的导出名；本题的核心数据变换只是逐字节减常数。
