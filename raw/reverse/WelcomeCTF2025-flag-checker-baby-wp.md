# Flag Checker Baby

## 题目简述

程序内嵌 21 个加密字节，并逐字符解密后与用户输入比较。加密关系在校验逻辑中可以直接恢复：

$$
\operatorname{enc}[i]=(\operatorname{flag}[i]\oplus0x5a)+7\pmod{256}
$$

因此逆变换为：

$$
\operatorname{flag}[i]=((\operatorname{enc}[i]-7)\bmod256)\oplus0x5a
$$

## 解题过程

从二进制的数据区或菜单的“Show encrypted bytes”选项取得数组：

```python
enc = [
    68, 47, 70, 42, 40, 48, 58, 62, 49, 61, 70,
    12, 62, 70, 59, 54, 12, 47, 70, 51, 46,
]
```

按校验函数的逆序先减 7，再与 `0x5a` 异或：

```python
enc = [
    68, 47, 70, 42, 40, 48, 58, 62, 49, 61, 70,
    12, 62, 70, 59, 54, 12, 47, 70, 51, 46,
]

flag = bytes((((value - 7) & 0xFF) ^ 0x5A) for value in enc)
print(flag.decode())
```

输出并提交：

```text
grey{simple_menu_rev}
```

逐字节逆变换与密文数组可由[公开参赛者题解](https://sl-lee.github.io/CTF-Writeups/NUS-Greyhats-Welcome-CTF-2025)交叉核对；所需运算顺序、常数和完整脚本已转写在正文中。

## 方法总结

- 核心技巧：从逐字节校验函数直接写出可逆变换，并按相反顺序执行逆运算。
- 识别信号：固定字节数组、逐字符循环、加常数和 XOR 常数的组合。
- 复用要点：减法要在 8 位范围内处理，使用 `& 0xff` 明确模拟无符号字节回绕；最后把结果送回菜单校验，而不是只看可打印性。
