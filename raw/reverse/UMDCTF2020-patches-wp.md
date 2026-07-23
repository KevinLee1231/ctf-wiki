# Patches

## 题目简述

程序包含 `ptrace` 反调试、PIN 检查和一个故意错误的版本号。真正的 flag 由两个 23 字节数组逐项 XOR 得到；既可以静态恢复，也可以修补版本常量让程序走到输出分支。

## 解题过程

源码把 `version_num` 初始化为 `0xf00d`，随后却要求它等于 `0x1337`。动态路线可以跳过 `ptrace` 失败分支，并把立即数改成 `0x1337`。静态路线更简单，直接异或两个数组：

```python
flag = bytes.fromhex(
    "18 01 bb d4 5d 01 52 b1 8e c2 e8 02 "
    "18 8e 8b 20 15 df 19 b0 58 0c 27"
)
key = bytes.fromhex(
    "2c 6d d7 8b 28 5e 35 81 fa b6 dc 5d "
    "7c be d4 11 66 80 69 84 2c 6f 4f"
)
print(bytes(left ^ right for left, right in zip(flag, key)).decode())
```

输出为：

```text
4ll_u_g0tt4_d0_1s_p4tch
```

套入格式：

```text
UMDCTF-{4ll_u_g0tt4_d0_1s_p4tch}
```

两个内置数组与源码输出格式都指向该文本；README 中未同步更新的哈希不作为覆盖程序证据的依据。

## 方法总结

当 flag 生成材料完整存在于二进制时，反调试和不可达分支不妨碍静态求值。补丁路线有助于理解题意，但 XOR 数组已经构成更短、可验证的恢复路径。
