# Evil Santa

## 题目简述

程序带有反调试检查，并要求输入长度为 40。随后按输入字符 ASCII 的奇偶性执行不同加法，再与硬编码字符串比较；逆向这段逐字符变换后还要进行 Base64 解码。

## 解题过程

调试时可以修改反调试返回值或跳过失败分支。核心比较目标为：

```text
XY5IU5TKNZvV][92PX:5Q[9YZ5Ts]XThQ7]zVJ2A
```

变换规则是：原字符为奇数时加 4，为偶数时加 2。增加量都是偶数，因此奇偶性不变，可以直接根据目标字符的奇偶性选择逆运算：

```python
target = "XY5IU5TKNZvV][92PX:5Q[9YZ5Ts]XThQ7]zVJ2A"
encoded = "".join(
    chr(ord(ch) - (4 if ord(ch) & 1 else 2))
    for ch in target
)
print(encoded)
```

恢复出：

```text
VU1EQ1RGLXtTYW50NV81MW5UX1RoYVRfM3YxTH0=
```

再做 Base64 解码：

```text
UMDCTF-{Sant5_51nT_ThaT_3v1L}
```

## 方法总结

反调试只是进入核心逻辑前的门槛，不应掩盖真正的数据流。由于加数保持奇偶性，本题的分支可以从变换后的字符反推；得到标准 Base64 后再解一层即可。
