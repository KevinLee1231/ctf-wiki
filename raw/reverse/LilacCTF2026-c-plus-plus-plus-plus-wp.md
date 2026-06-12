# c++++

## 题目简述

题目是 C# Native AOT 逆向，程序名为 Sentinel Guard。主程序读入 security token，用派生出的 key/IV 初始化 `XEngine`，再对输入做 16 字节分组变换，最后与固定十六进制密文比较：

```csharp
var engine = new XEngine(k, v);
byte[] result = engine.Transform(Encoding.UTF8.GetBytes(input));
string hexResult = BitConverter.ToString(result).Replace("-", "");

if (hexResult == "A20492152735B4F6ECBAA359DB64417BDF277A73B085666034CF38E748D8FBD4")
    Console.WriteLine("[+] ACCESS GRANTED.");
```

`XEngine` 内部包含 RS/MDS 表、40 个 round subkey 和 16 轮 Feistel-like 变换，结构特征明显接近 Twofish。题目目标就是对给定目标密文做逆变换，恢复能通过校验的 token。

## 解题过程

### 关键观察

源码中 `XEngine` 的结构与 Twofish 很接近：

- 使用 RS/MDS 相关表。
- 生成 40 个 round subkey。
- 16 轮 Feistel-like block transform。
- `F32` 负责 key-dependent S-box/MDS 混合。

key 和 IV 都不是用户输入，而是由固定字节数组异或 mask 得到：

```csharp
byte[] k = Derive(new byte[] { ... }, 0x41);
byte[] v = Derive(new byte[] { ... }, 0x31);
```

其中 `Derive` 只是逐字节异或：

```csharp
for (int i = 0; i < res.Length; i++)
    res[i] ^= mask;
```

### 求解步骤

由于 `Transform` 是确定性分组变换，且目标密文已知，可以直接实现逆变换：

1. 从 `Program.cs` 恢复 key/IV。
2. 根据 `Twofish.cs` 的 key schedule 生成相同的 `K[0..39]` 和 S 盒状态。
3. 对目标密文按 16 字节分块。
4. 反向执行 16 轮：撤销最后一轮 output whitening，再逆序还原轮函数和 swap。
5. 去掉零填充，得到原始 token。

如果实现正确，逆变换得到的明文重新送入 `XEngine.Transform` 后，输出应严格等于目标十六进制密文。

## 方法总结

- 核心技巧：识别被改名的 Twofish-like 结构，恢复固定 key 后对目标密文做逆变换。
- 识别信号：Native AOT/C# 程序里出现 RS、MDS、40 个 round key、16 轮分组操作，应联想到 Twofish。
- 复用要点：AI 或反编译器可能识别不出改名算法，但结构特征比类名可靠；先定位 key derivation、round key 和 whitening，再写逆变换。
