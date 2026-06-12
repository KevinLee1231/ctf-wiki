# D3Re

## 题目简述

题目是 C# 编写的 UWP 程序，但为了避免直接反编译 IL，使用了 .NET Native。校验入口在 `Tools.CheckFlag`：点击 Check 后，如果输入正确，文本框显示 `Good!!!`；否则显示 `No, wrong. Try again.`。这两个长字符串足够用来定位按钮事件和校验函数。

如果附件提供对应 PDB，可以辅助恢复符号；没有 PDB 时也可以通过 UI 字符串、按钮事件和加密常量定位主逻辑。

## 解题过程

校验器首先检查 flag 格式：`d^3ctf{GUID in lower case}`。

1. 长度必须为 44。

2. 前缀和后缀必须为 `d^3ctf{}`。

3. 花括号内部必须是小写 GUID。

GUID 会被转换成长度为 16 的 byte array，供后续校验使用。

前 8 个字节通过移位转换成大整数 `SUM`，随后检查：

$$
SUM \cdot 757726435240880506850652 \equiv 820856551661154796608770 \pmod {904559654629185507076703}
$$

`SUM mod 0x100000` 的逆元会作为下一步 AES key 的生成种子，IV 的种子固定为 15。

生成算法如下：

```csharp
public static byte[] GenKey(int cnt, long seed)
{

    byte[] r = new byte[cnt];
    for (int i = 0; i < cnt; i++)
    {
        seed = seed * 1103515245 + 12345;
        seed %= long.MaxValue;
        r[i] = ((byte)seed);
    }

    return r;
}
```

因此只需要生成 key 和 IV，再解密内嵌密文即可得到 flag：

```text
0x8f,0x7c,0x12,0x6b,0x07,0xd4,0x98,0x77,0x3b,0x5c,0x62,0x0b,0xac,0x96,0x13,0x96
```

## 方法总结

- 核心技巧：UWP/.NET Native 题不要只依赖 IL 反编译；UI 字符串、事件处理函数、PDB 符号和加密常量都可以作为定位点。
- 解题路径：先验证 flag 格式和 GUID 解析，再解前 8 字节对应的模方程，最后用 `SUM mod 0x100000` 的逆作为 AES key seed，IV seed 固定为 15。
- 复用要点：自定义 LCG 派生 key/IV 的题，重点是还原 seed 来源和字节截断方式，真正的 AES 解密通常只是最后一步。
