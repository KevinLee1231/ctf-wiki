# Santa's Mysterious Box

## 题目简述

程序逐字符处理输入并与硬编码字符串比较。每个输入字节在比较前减去 5，因此只要对目标串逐字节加 5，就能恢复正确输入。

## 解题过程

反编译得到比较目标：

```text
PH?>OA(vN/io/Zb<q.Zt+pZKm.io0x
```

逆运算如下：

```python
target = b"PH?>OA(vN/io/Zb<q.Zt+pZKm.io0x"
plain = bytes((value + 5) & 0xff for value in target)
print(plain.decode())
```

输出为：

```text
UMDCTF-{S4nt4_gAv3_y0u_Pr3nt5}
```

## 方法总结

逐字节加减常数是最直接的可逆变换。恢复时应从比较端向输入端倒推，使用相反运算，并以程序检查的实际字节串为准，不需要暴力枚举。
