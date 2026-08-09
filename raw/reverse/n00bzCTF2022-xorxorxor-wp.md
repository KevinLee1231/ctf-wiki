# xorxorxor

## 题目简述

程序把输入的第 $i$ 个字节与下标 $i$ 异或，再和固定字节串比较。异或是自反运算，因此对目标串执行同样的逐下标异或即可恢复原输入。

## 解题过程

比较目标为：

```text
n12a~~~7zV;xSyf<Os!``4k
```

恢复脚本如下：

```python
target = b"n12a~~~7zV;xSyf<Os!``4k"
flag = bytes(value ^ index for index, value in enumerate(target))
print(flag.decode())
```

输出：

```text
n00bz{x0r_1s_th3_b3st!}
```

## 方法总结

本题不需要爆破。若校验逻辑是 $y_i=x_i\oplus i$，则直接利用 $x_i=y_i\oplus i$ 逆变换；同时要按字节处理，避免字符串编码引入额外变化。
