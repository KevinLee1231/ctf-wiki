# Intro to AES 2

## 题目简述

服务以固定随机 IV 和密钥执行 AES-CBC，加密内容为 `iv || flag` 并使用 PKCS#7 补位。解密接口不返回明文，却会根据补位是否正确分别响应成功或失败，因而构成 CBC padding oracle。目标是逐字节恢复 31 字节 flag。

## 解题过程

CBC 解密满足 $P_i=D_K(C_i)\oplus C_{i-1}$。记中间值为 $I_i=D_K(C_i)$，只要修改前一密文块，就能控制当前明文块：

$$P_i'=I_i\oplus C_{i-1}'$$

从块尾向前枚举。若要让最后一个字节成为合法补位 `0x01`，遍历前一块对应字节 $x$，直到 oracle 返回成功，此时：

$$I_i[-1]=x\oplus 1$$

进而得到原明文字节：

$$P_i[-1]=I_i[-1]\oplus C_{i-1}[-1]$$

恢复第 $k$ 个尾部字节时，先把已经求出的后缀调整为值 $k$，再枚举当前字节。核心逻辑如下：

```python
def recover_block(prev, cur, oracle):
    crafted = bytearray(prev)
    intermediate = bytearray(16)
    plain = bytearray(16)

    for pad in range(1, 17):
        pos = 16 - pad
        for j in range(pos + 1, 16):
            crafted[j] = intermediate[j] ^ pad
        for guess in range(256):
            crafted[pos] = guess
            if oracle(bytes(crafted) + cur):
                intermediate[pos] = guess ^ pad
                plain[pos] = intermediate[pos] ^ prev[pos]
                break
    return bytes(plain)
```

对目标的两个 flag 分组依次执行该过程。服务在 flag 前加密了 IV，因此首个实际 flag 块也有一个可修改的前置密文块；恢复后去除 PKCS#7 补位即可得到：

```text
grey{p4dd1ng_0r4cl3_pr0_g4m1ng}
```

## 方法总结

padding oracle 的本质是服务泄露了“解密后的补位是否合法”这一位信息。CBC 的异或可控性把这位信息转化为逐字节的明文恢复能力。修复时应使用认证加密，并让所有解密失败走不可区分的错误路径。
