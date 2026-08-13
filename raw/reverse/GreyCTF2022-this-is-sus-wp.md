# GreyCTF2022 - this-is-sus

## 题目简述

附件是 UPX 打包且缺少符号的 32 位 ELF，包含大量库代码。真正的用户逻辑动态拼出 RC4 密钥，解密全局密文后又对每个字节的高、低半字节做自定义重排；两层变换都可逆。

## 解题过程

先用 UPX 解包，减少入口壳代码干扰：

```bash
upx -d sus -o sus-unpacked
```

题目提示“并非所有调用都出现在交叉引用中”，因此从输入、全局密文写入和 RC4 状态初始化等数据流定位用户函数，而不是只依赖调用图。还原运行时拼接的 key 后，按标准 RC4 KSA/PRGA 解密全局缓冲区；再逆转编码函数对两个 nibble 的交换/映射。

```python
stage1 = rc4(key, ciphertext)
plain = bytes(unshuffle_nibbles(x) for x in stage1)
print(plain.decode())
```

仓库的 `solve.c` 也以相反顺序调用同一组可逆函数，输出：

```text
flag{<3_N3v3r_G0nn4_G1v3_u_Up_<3}
```

## 方法总结

解包后仍要区分静态链接库代码与用户逻辑。没有可靠 xref 时，可追踪输入缓冲区、全局密文和显著常量的读写。RC4 加解密相同，但后处理未必是自身的逆，必须按源码精确实现 nibble 级逆映射。
