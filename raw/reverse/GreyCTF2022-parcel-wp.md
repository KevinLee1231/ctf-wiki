# GreyCTF2022 - Parcel

## 题目简述

题目给出一个带大量无关函数的 ELF。程序要求输入三个指定函数 `h12`、`t80`、`g20` 的地址，正确后用四字节密钥解开内置密文。关键不是逐个阅读数千个函数，而是利用符号表定位目标。

## 解题过程

文件保留符号，直接查询即可：

```bash
nm parcel | grep -E ' (h12|t80|g20)$'
# 也可使用：objdump -t parcel
```

仓库二进制中三者地址分别为 `0x403215`、`0x406891`、`0x402e21`，换成程序要求的十进制后输入。通过校验后，程序设置密钥字节为 `key` 并对静态密文循环异或；也可以直接从源码/反编译结果复现：

```python
plain = bytes(c ^ b"key"[i % 3] for i, c in enumerate(ciphertext))
print(plain.decode())
```

得到：

```text
grey{d1d_y0u_us3_nm_0r_objdump_0r_gdb_0r_ghidra_0r_rizin_0r_ida_0r_binja?}
```

## 方法总结

面对大量诱饵函数，先检查 ELF 是否剥离符号、是否开启 PIE，以及题目要求的是符号地址还是运行时地址。保留符号时，`nm`/`objdump -t` 比手工反编译更直接；若开启 PIE，则还要结合加载基址换算。
