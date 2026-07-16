# week2re3

## 题目简述

ELF 在 main 之前通过 .init_array 构造函数执行反调试和代码解密。构造函数调用 ptrace 检测调试器；通过检测后才会解密地址 0x401216 附近的函数。解密后的逻辑是标准 RC4，密钥和密文都可从程序中提取。

## 解题过程

先定位先于 main 执行的 sub_401bd0。动态调试时可在 ptrace 返回后把 rax/eax 改为 0，或在副本中修补条件跳转，使程序进入正常解密分支。继续执行到 0x401216 后重新分析该区域，可以看到 RC4 的两个典型阶段：

- 初始化 0 到 255 的 S 盒，并按密钥循环置换，这是 KSA。
- 每轮更新 i、j、交换 S 盒元素并异或一个密钥流字节，这是 PRGA。

程序内置密钥为 0xdeadbeef，实际密文为前 42 个非零字节；数组末尾两个 0 只是 C 字符串终止和填充。用等价脚本解密：

~~~python
key = b"0xdeadbeef"
cipher = bytes([
    0xBA, 0x9E, 0x78, 0xD9, 0xC3, 0x68, 0x5E, 0xC9,
    0x0B, 0x7E, 0x21, 0x79, 0x4F, 0x13, 0x71, 0xAA,
    0x5C, 0x2C, 0x64, 0xB0, 0x09, 0x96, 0x73, 0xD5,
    0x34, 0xC4, 0x5C, 0xBA, 0xE2, 0xF9, 0x6D, 0xC7,
    0x3D, 0x52, 0x78, 0x3A, 0x20, 0xEF, 0x5D, 0xCD,
    0xDA, 0xE3,
])

s = list(range(256))
j = 0
for i in range(256):
    j = (j + s[i] + key[i % len(key)]) & 0xff
    s[i], s[j] = s[j], s[i]

i = j = 0
plain = bytearray()
for value in cipher:
    i = (i + 1) & 0xff
    j = (j + s[i]) & 0xff
    s[i], s[j] = s[j], s[i]
    plain.append(value ^ s[(s[i] + s[j]) & 0xff])

print(plain.decode())
~~~

输出为：

~~~text
flag{25f2d963-27a4-402c-b40c-62d682cf1913}
~~~

## 方法总结

- 核心方法：绕过 ptrace 反调试，让程序解密隐藏代码，再识别并复现 RC4 KSA/PRGA。
- 识别特征：main 前构造函数、ptrace 检查、自修改代码，以及 256 字节 S 盒的两阶段置换。
- 注意事项：不要在代码尚未解密时直接反编译目标地址；RC4 加解密相同，但密文长度必须按程序实际使用的 strlen 结果确定。
