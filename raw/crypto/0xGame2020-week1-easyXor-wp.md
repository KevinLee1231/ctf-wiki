# week1easyXor

## 题目简述

加密脚本先在 flag 末尾追加已知字符 `^`，再把每个相邻字符异或：

$$
c_i = m_i \oplus m_{i+1}
$$

由于最后一个明文字节已知，可以从密文末尾向前递推全部明文，而不需要爆破。

```python
flag += "^"
cipher = []
for i in range(len(flag) - 1):
    cipher.append(ord(flag[i]) ^ ord(flag[i + 1]))
```

## 解题过程

由异或自反性可得：

$$
m_i = c_i \oplus m_{i+1}
$$

令最后一个已知字节为 `^`，逆序遍历密文即可：

```python
cipher = [
    72, 63, 38, 12, 8, 30, 30, 6, 82, 4, 84, 88, 92, 7, 79, 29,
    8, 90, 85, 26, 25, 87, 80, 10, 20, 20, 9, 4, 80, 73, 31, 5,
    82, 0, 1, 92, 0, 0, 94, 81, 4, 85, 27, 35,
]

next_byte = ord("^")
plain_reversed = []

for value in reversed(cipher):
    current = value ^ next_byte
    plain_reversed.append(current)
    next_byte = current

flag = bytes(reversed(plain_reversed)).decode()
print(flag)
```

输出为：

```text
0xGame{ec15a9eb-08b7-4c39-904d-27eed888f73f}
```

## 方法总结

- 核心技巧：利用已知尾字符，从相邻异或链的末端反向递推。
- 识别信号：密文元素满足相邻明文异或，且至少有一个端点明文已知。
- 复用要点：必须逆序恢复；若没有已知端点，则任取一个首字节会产生一整条候选链，需要再用格式约束筛选。
