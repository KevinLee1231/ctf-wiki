# Gates

## 题目简述

WelcomeCTF2021 的 Gates 在浏览器端 JavaScript 中实现四道口令检查。所有校验和 flag 拼接逻辑都下发到客户端，因此无需攻击服务器，只要逆向 `gates.js` 中的四个函数。

## 解题过程

Gate 1 直接比较字符串，口令是：

```text
1ts
```

Gate 2 要求 6 字节，并让每个字符与固定单词中的某个字符异或后等于目标常量。逐位逆运算 `key[i] = reference ^ target`，得到：

```text
c0m1ng
```

Gate 3 设两个字符码为 $x>y$，并要求：

$$
x+y=164,\qquad xy=5568.
$$

解二次方程或枚举 ASCII，可得 $x=116$、$y=48$，即 `t0`。

Gate 4 对四个字符分别执行 8 位循环左移，位数为 `[2,3,4,5]`，目标为 `[201,129,214,102]`。枚举可打印 ASCII 或执行逆向循环右移：

```python
rotations = [2, 3, 4, 5]
targets = [201, 129, 214, 102]

answer = []
for rotation, target in zip(rotations, targets):
    for value in range(0x20, 0x7f):
        rol = ((value << rotation) & 0xff) | (value >> (8 - rotation))
        if rol == target:
            answer.append(chr(value))
            break

print("".join(answer))
```

输出为 `r0m3`。页面按通过顺序以 `_` 连接四个口令，得到：

```text
greyhats{1ts_c0m1ng_t0_r0m3}
```

## 方法总结

客户端校验不能保护秘密：用户可以完整读取校验函数、常量和 flag 组装规则。四关分别用直接比较、异或、整数方程和循环移位包装字符串，但每一步都可直接逆运算或在很小的 ASCII 空间枚举。
