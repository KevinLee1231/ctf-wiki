# d3hell

## 题目简述

题目由 exe 和 dll 组成。exe 表面上做大整数分解并最终按两个因子异或输出 flag；dll 在加载时通过 DllMain 执行，检测调试/patch 行为并给 exe 注入真实或伪造输入。关键是识别 dll 中 TEA 解密出的真实大数，避免被 fake flag 和耗时 Pollard-rho 分解干扰。

## 解题过程

这不是本次 CTF 中很难的逆向题。附件包含一个 exe 文件和一个 dll 文件，下面分别分析。

**exe 部分**

exe 的主要工作是计算一个大数的素因子。

```
  sub_403730(argc, argv, envp);
  puts("Take a rest. This is a easy RE problem!");
  Sleep(0xBB8u);
  puts("The flag will show up after a few minutes.");
  Sleep(0xBB8u);
  puts("Just be patient!");
  Sleep(0xBB8u);
  *(double *)&v3 = sub_4014F0(); // Get input
  v14 = v3;
  if ( v3 == 0 )
    return 0;
  unk_4099F0 = 0i64;
  *((_QWORD *)&unk_4099F0 + 1) = 0i64;
  v13 = v14;
  v12[0] = 120i64;
  v12[1] = 0i64;
  sub_40216A(&v13, v12); // Prime factorization
```

程序使用的算法是 `Pollard_rho`。简单来说，这是一种可用的素因子分解方法，但程序逻辑不清晰，分析体验很差。完整理解整个素因子计算过程并不容易，可以从以下几个方向入手：

把输入改成较小数字，观察对应输出。

- 修补掉 SMC 和反调试残留后直接等待，但这大约需要半小时。

- 直接猜测。

- 使用数学和算法知识完整分析，不推荐。

exe 中其他函数并不重要，只需要考虑三点：while 中的 `sleep`（与调试相关）、一个只返回常量的函数（实际上不会造成太多延迟）、exe 本身没有反调试。即使 patch 掉 sleep，大部分时间仍然消耗在算法本身。

素因子计算完成后，程序会以 `antd3ctf{(factor1 after xor)+(factor2 after xor)}` 的形式打印 flag。

### dll 部分

dll 分析并不困难。64-bit 到 32-bit 的切换可能会影响 gdb 动态调试，但如果熟悉汇编，恢复代码不是大问题。这些 bytecode 会逐字节相互异或，恢复代码如下：

```
    uint32_t v1[2]={0xC29319B7,0xA47CC631};
    uint32_t v2[2]={0x45CFAC9E,0xC39089A6};
    uint32_t const k[4]={0x00114514,0x01919810,0x24270047,9};
__int128tmp=0;
    int c=0;
    unsigned int r=32;
    int i = 0;
    unsigned char *ans;
    ans = (unsigned char *) 0x0000000000401F13;
    if (!IsDebuggerPresent() && c == 0){
        uint32_t v1[2]={0xA532267E,0x8EFB0F27};
        uint32_t v2[2]={0x3C5F791A,0x1A38CC77};
        for(i=0; i<40;i++)
        {
            re_table[i] = 0x0;
        }
    }
    decipher(r, v1, k);
    decipher(r, v2, k);
    tmp += (__int128)v1[0] << 96;
    tmp += (__int128)v1[1] << 64;
    tmp += (__int128)v2[0] << 32;
    tmp += (__int128)v2[1];
    for(i=0; i<40;i++)
    {
        *(unsigned char *)(i + 0x00405020) = flag[i];
    }
    for(i=0; i<40;i++)
    {
        *(unsigned char *)(i + 0x00405060) = re_table[i];
    }
```

这段代码会检测 gdb 以及 exe 中的 patch（while 中的 `sleep` 函数）。如果检测到修改，就会给出 fake flag。真实数字经过 TEA 加密编码，解出为 `698740305822331500978964939673142241`。

如果已经理解 `d3hell.exe` 的逻辑，可以直接把这个数字输入 FactorDB。

也可以直接 patch dll，确保正确输入被发送给 exe，然后等待 flag 输出。

### 技巧

因为 `DllMain` 中 `fdwReason == 1`：

```
BOOL __stdcall DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
__int64v3; // rcx
  GetModuleHandleA("d3runtime.dll");
  if ( fdwReason == 1 )
    61FC1628(v3);
  return 1;
}
```

所以函数 `61FC1628` 只会在 dll 被加载时调用。可以直接让 gdb attach 到进程，拿到正确数字和表，而不必完整分析 dll。

## 方法总结

- 核心技巧：exe/dll 联动分析、DllMain 加载时机、反调试/fake flag 识别、TEA 解密真实输入、Pollard-rho 或 FactorDB 分解大数。
- 识别信号：主程序逻辑看似耗时但外部 dll 在加载时修改数据时，先查 DllMain 和全局表初始化；输出异常慢或 fake flag 时要怀疑反调试分支。
- 复用要点：可以动态 attach 到 dll 初始化之后直接取真实数字和表，减少完整逆 Pollard-rho 与混淆代码的成本。
