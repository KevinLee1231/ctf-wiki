# Change

## 题目简述

题目是关闭编译优化的 C++ 程序，通过类成员、函数注册/调用封装和运算符重载隐藏实际变换。`main` 读取 `plaintext`，调用 `Encryptor::operator+` 生成 `result`，再与 24 字节常量数组逐项比较。恢复类结构和重载函数后可知：偶数位置先与循环密钥异或再加 10，奇数位置只做异或，逆运算即可得到 flag。

## 解题过程

加载类型信息后，主逻辑可整理为：

```cpp
Encryptor encryptor("am2qasl");
std::string plaintext;
std::cin >> plaintext;

std::string result = encryptor + plaintext;
for (int i = 0; i < 24; ++i) {
    if (result[i] != ciphertext[i]) {
        std::cout << "sry,try again...";
        return 0;
    }
}
std::cout << "Congratulations!";
```

比赛附件没有直接提供完整类结构时，需要在反编译器中手工恢复 `std::string` 参数、`this` 指针和返回对象。不要被 `operator+` 的名字误导：它不是字符串拼接，而是逐字节加密函数。

重载函数遍历输入，根据下标奇偶注册不同的处理函数，再以 `key[i % key.length()]` 为参数执行。结合两个被注册函数的实际行为，可写出加密关系：

$$
c_i=
\begin{cases}
(p_i\oplus k_i)+10,&i\equiv0\pmod2,\\
p_i\oplus k_i,&i\equiv1\pmod2.
\end{cases}
$$

因此逆变换为：

$$
p_i=
\begin{cases}
(c_i-10)\oplus k_i,&i\equiv0\pmod2,\\
c_i\oplus k_i,&i\equiv1\pmod2.
\end{cases}
$$

题目中的密文与密钥为：

```text
c = 19, 10, 93, 28, 14, 8, 35, 6,
    11, 75, 56, 34, 13, 28, 72, 12,
    102, 21, 72, 27, 13, 14, 16, 79

key = am2qasl
```

完整解密脚本如下：

```python
ciphertext = [
    19, 10, 93, 28, 14, 8, 35, 6,
    11, 75, 56, 34, 13, 28, 72, 12,
    102, 21, 72, 27, 13, 14, 16, 79,
]
key = b"am2qasl"

plaintext = bytearray()
for i, value in enumerate(ciphertext):
    key_byte = key[i % len(key)]
    if i % 2 == 0:
        plaintext.append(((value - 10) & 0xFF) ^ key_byte)
    else:
        plaintext.append(value ^ key_byte)

print(plaintext.decode())
```

输出为：

```text
hgame{ugly_Cpp_and_hook}
```

官方截图中的字体容易把密钥末尾的小写字母 `l` 看成数字 `1`。PDF 文本层和解密结果都表明正确密钥是 `am2qasl`；若误写成 `am2qas1`，输出会在多个位置变成乱码式字符。

## 方法总结

- C++ 逆向先恢复对象边界、成员类型和运算符原型，可显著减少反编译器生成的临时变量噪声。
- 运算符重载只是语法外壳，必须进入 `operator+` 查看真实语义，不能按名称假定是加法或拼接。
- 函数注册、hook 或回调封装会遮蔽具体运算；应追踪最终被调用函数的行为，而不是依赖 `xor1`、`xor2` 等标签。
- 逆运算顺序必须与加密相反：偶数位先减 10，再 XOR；把顺序写反不会恢复原文。
- 对字体易混淆的密钥字符，应同时用 PDF 文本层和输出格式校验，避免把 `l` 抄成 `1`。
