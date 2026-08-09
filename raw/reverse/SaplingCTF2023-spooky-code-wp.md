# Spooky Code

## 题目简述

程序把一大段 data 复制到按页分配的 RWX 区域并执行。有效代码和编码后的 flag 字节以 0x20 为步长散布在 ELF 数据中，运行时链式异或恢复后续字符，形成“代码段很小、逻辑藏在数据段”的效果。

## 解题过程

官方解法无需完整仿真自修改代码，直接按发布 ELF 的文件偏移提取。第一个字符位于 0x1062，后续字节从 0x1080 到 0x17ac 每隔 0x20 取一个：

~~~python
with open("spooky", "rb") as f:
    b = f.read()

flag = [b[0x1062]]
flag += list(b[0x1080:0x17ac:0x20])

for i in range(1, len(flag)):
    flag[i] ^= flag[i - 1]

print(bytes(flag).decode())
~~~

恢复结果：

~~~text
maple{sP0oKy_s3lf_m0d1fy1ng_c0de_4pp34r1ng_0ut_0f_n0wh3r3}
~~~

## 方法总结

ELF 的可执行逻辑不一定只在 .text 中。看到 mmap RWX、memcpy 数据并间接调用时，应检查数据段的规律和运行时写入。固定文件偏移只适用于给定附件；换版本时应通过段表或字节签名重新定位，不能照搬数值。
