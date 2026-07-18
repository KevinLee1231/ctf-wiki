# week2reading_asm_is_based

## 题目简述

题目给出一段 C++ 字符串处理程序的反汇编。循环从左到右修改除末字节外的每个字符，当前字符先与下标异或、减去 `0x40`，再与尚未修改的下一个字符异或：

$$
c_i=((m_i\oplus i)-0x40)\oplus m_{i+1}
$$

最后一个字符不参与变换，即 $c_{L-1}=m_{L-1}$。由于每个位置依赖右侧字符，逆运算必须从末尾向前进行。

## 解题过程

反汇编中的核心指令可归纳为：

```asm
movzx eax, byte ptr [flag + i]
xor   eax, i
sub   eax, 40h
movzx ebx, byte ptr [flag + i + 1]
xor   eax, ebx
mov   byte ptr [flag + i], al
```

它对应的源码循环为：

```cpp
for (int i = 0; i < flag.size() - 1; ++i) {
    flag[i] = ((flag[i] ^ i) - 0x40) ^ flag[i + 1];
}
```

移项可得逆变换：

$$
m_i=((c_i\oplus m_{i+1})+0x40)\oplus i
$$

对程序输出从右向左原地恢复：

```python
cipher = r"ElayA@F{dIGBtOxxEzmNVZOMVL\Jb}oh"
data = bytearray(cipher.encode())

for index in range(len(data) - 2, -1, -1):
    data[index] = ((data[index] ^ data[index + 1]) + 0x40) ^ index

print(data.decode())
```

输出为：

```text
GBoLvsvvJffkbZXnYLgXEGHQKEPVGyYh
```

将恢复结果重新代入正向循环，可以逐字节得到原密文 `ElayA@F{dIGBtOxxEzmNVZOMVL\Jb}oh`，由此完成验证。

## 方法总结

- 核心技巧：把汇编归纳为相邻字符递推式，再按依赖方向逆序求解。
- 识别信号：循环原地修改数组，当前项同时依赖下标和右侧尚未修改的元素。
- 复用要点：先判断下一项在正向循环中是否已被修改；逆推方向错误会导致后续所有字节连锁出错。
