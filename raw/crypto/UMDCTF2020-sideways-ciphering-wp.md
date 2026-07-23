# Sideways Ciphering

## 题目简述

远程服务使用 AES-CBC 加密并反馈 PKCS#7 填充是否合法。虽然没有直接泄露密钥，但“填充正确/错误”形成了 padding oracle，可以逐字节恢复每个密文块对应的中间值与明文。

## 解题过程

对相邻密文块 $C_{i-1}$、$C_i$，CBC 解密满足：

$$
P_i=D_K(C_i)\oplus C_{i-1}
$$

记中间值 $I_i=D_K(C_i)$。为了恢复块尾字节，把伪造前块的最后一个字节从 0 到 255 枚举；当服务返回填充有效时，有：

$$
I_i[-1]\oplus C'_{i-1}[-1]=1
$$

由此得到 $I_i[-1]$，再与原始 $C_{i-1}[-1]$ 异或得到明文字节。之后把已恢复位置统一调整为填充值 2、3，依次向前推进：

```python
def recover_block(previous, current, oracle):
    intermediate = bytearray(16)
    plaintext = bytearray(16)

    for pad in range(1, 17):
        pos = 16 - pad
        forged = bytearray(previous)
        for j in range(pos + 1, 16):
            forged[j] = intermediate[j] ^ pad

        for guess in range(256):
            forged[pos] = guess
            if oracle(bytes(forged) + current):
                intermediate[pos] = guess ^ pad
                plaintext[pos] = intermediate[pos] ^ previous[pos]
                break
        else:
            raise RuntimeError("未找到有效填充")

    return bytes(plaintext)
```

逐块恢复并去除 PKCS#7 填充后得到：

```text
UMDCTF-{s1d3_ch@nn3l_0p3n}
```

## 方法总结

CBC 本身并不会隐藏解密端的差异化错误。只要攻击者能稳定区分填充是否合法，就能把响应当作一比特 oracle，逐字节求出中间值；修复时应统一错误响应，并在认证通过前拒绝处理密文。
