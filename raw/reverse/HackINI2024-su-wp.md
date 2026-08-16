# HackINI2024 su

## 题目简述

题目提供一个原生程序，要求恢复 16 字节密码。正确密码会触发取得 shell 的分支，flag 格式为 `shellmates{password}`。密码检查被拆成逐字节整数约束，其中最后几位还使用按位与和右移表达式。

## 解题过程

静态反编译或在最终比较处动态观察，可以整理出以下约束：

```text
password[0]  == 36
password[1]  == 85
password[2]  == 80
password[3]  == 51
password[4]  == 82
password[5]  == 95
password[6]  == 36
password[7]  == 51
password[8]  == 67
password[9]  == 82
password[10] == 51
password[11] == 84
password[12] == 95
(password[13] & 0x1fffffff) == 0x6b
(password[14] >> 2) == 12
password[15] == 89
```

前 13 项和最后一项直接转为 ASCII。第 13 位的低值约束得到 `0x6b`，即 `k`；第 14 位右移 2 位等于 12，官方预期可打印字符是 ASCII 51，即 `3`。逐位还原：

```python
values = [36, 85, 80, 51, 82, 95, 36, 51, 67, 82, 51, 84, 95]
password = "".join(map(chr, values)) + "k3Y"
print(password)
```

得到密码：

```text
$UP3R_$3CR3T_k3Y
```

按题目指定格式，flag 为：

```text
shellmates{$UP3R_$3CR3T_k3Y}
```

## 方法总结

逐字节约束题不需要盲猜完整字符串。先把每个比较统一成字符码，再分别处理掩码和移位：掩码约束在单字节输入上通常直接给出字符，右移约束则可能对应一个小范围，需结合可打印字符和 flag 语义选择并回代验证。最终密码的每一位都能由检查逻辑解释。
