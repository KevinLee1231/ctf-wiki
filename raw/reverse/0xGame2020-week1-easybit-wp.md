# week1easybit

## 题目简述

附件是 64 位 Windows PE。校验逻辑对每个 8 位明文字节执行循环右移 3 位，再与内置数组比较。循环移位不会丢失比特，因此逆变换是对密文字节循环左移 3 位。

设加密结果为 $c$、明文字节为 $m$：

$$
c=\operatorname{ROR}_8(m,3)
$$

则：

$$
m=\operatorname{ROL}_8(c,3)=((c\ll3)\mathbin{\&}255)\mathbin{|}(c\gg5)
$$

## 解题过程

从反编译结果中抄出比较数组，逐字节执行逆向循环移位：

```python
cipher = [
    0x06, 0x0F, 0xEC, 0x2C, 0xAD, 0xAC, 0x6F, 0x4E, 0x66, 0xEB,
    0x26, 0x6E, 0xEB, 0xCE, 0xAC, 0x4E, 0xE6, 0xEB, 0x8D, 0xCD,
    0x8E, 0x66, 0x4E, 0xAC, 0x6E, 0x8E, 0x2D, 0xCD, 0x27, 0xAF,
]


def rol8(value, count):
    count %= 8
    return ((value << count) & 0xFF) | (value >> (8 - count))


flag = bytes(rol8(value, 3) for value in cipher)
print(flag.decode())
```

输出为：

```text
0xgame{r3_1s_ver7_lnt3restin9}
```

## 方法总结

- 核心技巧：用相反方向、相同位数的循环移位恢复每个字节。
- 识别信号：反编译代码同时出现左移、右移并把两部分合并，且结果限制在 8 位。
- 复用要点：普通右移会丢弃低位，循环右移不会；逆运算必须限定字长并对左移结果掩码。
