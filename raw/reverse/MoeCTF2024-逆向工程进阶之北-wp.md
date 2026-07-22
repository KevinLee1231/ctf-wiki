# 逆向工程进阶之北

## 题目简述

程序把 flag 按 32 位无符号整数处理，依次执行乘法、加法和异或。解密时除了逆序撤销异或与加法，还要在模 $2^{32}$ 的整数环上求乘数 `0xccffbbbb` 的逆元。

## 解题过程

`DWORD` 的所有运算都会截断到 32 位，因此模数是 $2^{32}=0xffffffff+1$，不是 `0xffffffff`。因为

$$
\gcd(0xccffbbbb,2^{32})=1,
$$

乘法逆元存在，可直接用 Python 求得：

```python
from Crypto.Util.number import inverse

inv = inverse(0xCCFFBBBB, 1 << 32)
print(inv)  # 2371998067
assert (0xCCFFBBBB * inv) % (1 << 32) == 1
```

按加密的相反顺序逐项还原，并依靠 `uint32_t` 自动执行模 $2^{32}$ 截断：

```cpp
#include <cstdint>
#include <iostream>

int main() {
    uint32_t data[12] = {
        0xb5073388, 0xf58ea46f, 0x8cd2d760, 0x7fc56cda,
        0x52bc07da, 0x29054b48, 0x42d74750, 0x11297e95,
        0x5cf2821b, 0x747970da, 0x64793c81, 0x00000000
    };

    for (int i = 0; i < 11; ++i) {
        data[i] ^= 0xDEADBEEF + 0xD3906;
        data[i] -= 0xDEADC0DE;
        data[i] *= 2371998067U;
    }

    std::cout << reinterpret_cast<unsigned char *>(data) << '\n';
}
```

输出为：

```text
moectf{c5f44c32-cbb9-444e-aef4-c0fa7c7a6b7a}
```

## 方法总结

逆向固定宽度整数算法时，要把溢出视为模运算。乘法只有在乘数与模数互素时才可逆；求出逆元后，仍必须严格按“异或、减法、乘逆元”的逆序撤销原变换。原稿中的在线逆元计算器只是通用工具链接，正文已有完整计算方法，因此不再保留。
