# Organised Person

## 题目简述

附件是一个 Windows PE flag checker。外层程序不是直接校验输入，而是用硬编码密钥 XOR 解密一段嵌入 PE，将其映射到内存、修改页面权限，再解析其中的 `check_flag` 导出。内层校验器才用第二个循环密钥解密预置 flag，因此需要连续还原两层。

## 解题过程

入口函数调用一个加载例程。该例程对约 `0x3800` 字节的嵌入载荷循环 XOR：

```python
outer_key = b"grey{l00k_aT_mE_im_da_fL4g_wAit_y0u_aRe_igNor1nG_me_T_T}"
payload = bytes(value ^ outer_key[index % len(outer_key)]
                for index, value in enumerate(encrypted_payload))
open("unpacked.bin", "wb").write(payload)
```

解密结果是另一个 PE。外层随后使用 `VirtualAlloc` 分配内存、复制并重定位映像，再用 `VirtualProtect` 把页面改为可执行，最后定位 `check_flag` 导出并调用。这也解释了题目中 “organise” 和 “packing” 的提示：核心障碍是自定义内存打包器，而不是普通压缩包密码。

打开 `unpacked.bin` 分析 `check_flag`，可恢复内层密文字节数组与循环密钥：

```python
encrypted = [
    0x00, 0x00, 0x00, 0x00, 0x00, 0x04, 0x05, 0x17, 0x01, 0x37,
    0x3c, 0x17, 0x04, 0x12, 0x32, 0x59, 0x2c, 0x20, 0x17, 0x5e,
    0x21, 0x38, 0x32, 0x02, 0x59, 0x63, 0x43, 0x1d, 0x00, 0x4b,
    0x68, 0x49, 0x0e, 0x28, 0x1c, 0x3a, 0x10, 0x15, 0x3a, 0x0c,
    0x7c, 0x37, 0x1f, 0x78, 0x2d, 0x50, 0x29, 0x00, 0x53, 0x6c,
    0x09,
]
key = b"grey{ehHhhH_aM_i_tH1s_sl00py_;-;}"
```

对第 $i$ 个字节执行：

$$
P_i=C_i\oplus K_{i\bmod |K|}
$$

```python
plaintext = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(encrypted)
)
print(plaintext.decode())
```

解密得到：

```text
grey{am_i_tHe_m0sT_oRgAniS3d_pErsOn_in_d4_w0r1d_:3}
```

外层打包器细节由附件反汇编与参赛者复盘相互印证；后者可见 [SL 的 WelcomeCTF 2025 题解](https://sl-lee.github.io/CTF-Writeups/NUS-Greyhats-Welcome-CTF-2025)。正文已经概括其关键调用链，链接仅保留为反汇编证据来源。

## 方法总结

- 核心技巧：先从外层自定义加载器解出嵌入 PE，再分析内层导出函数中的重复密钥 XOR。
- 识别信号：`VirtualAlloc`、`VirtualProtect`、手工定位导出和大块循环 XOR 共同指向内存解包；内层密文前五字节为零，则与明文、密钥同为 `grey{` 一致。
- 复用要点：分析自定义壳时应区分“解包密钥”和“业务数据密钥”；已知 flag 前缀可同时验证解密结果、字节序和循环起点。
