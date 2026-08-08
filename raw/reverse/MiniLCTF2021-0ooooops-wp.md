# MiniLCTF 2021 - 0ooooops

## 题目简述

附件是 32 位 Windows PE。主函数只检查 flag 格式，随后故意触发访问违例和除零异常；真正的逐字节校验分别藏在 Vectored Exception Handler（VEH）与 Structured Exception Handling（SEH）过滤函数中。TLS 回调在程序入口前注册 VEH，因此只沿主函数控制流不会看到完整检查逻辑。

## 解题过程

格式检查要求总长度为 72，前缀为 `miniLctf{`，末尾下标 71 是 `}`。随后 `EBX` 指向输入缓冲区，异常处理器从 `EBX+9` 取得花括号内的数据。

访问违例处理器校验偶数位置：

$$
enc1_i=((flag_{2i}\oplus55)+4)\oplus66.
$$

除零异常的 SEH 过滤器校验奇数位置：

$$
enc2_i=((flag_{2i+1}\oplus77)-4)\oplus0x13\oplus(EIP\mathbin{\&}0xff).
$$

触发该异常时，`EIP` 低字节为 `0x4b`。分别逆运算后交错两个字符序列：

```python
enc1 = [
    16, 4, 24, 11, 24, 16, 4, 21, 11, 5, 31, 46, 33, 46, 72, 21,
    6, 46, 17, 69, 5, 62, 46, 24, 21, 72, 46, 69, 33, 31, 10,
]
enc2 = [
    33, 86, 32, 45, 125, 86, 71, 45, 98, 112, 125, 109, 45, 110, 71,
    33, 98, 124, 114, 97, 32, 71, 121, 71, 69, 124, 68, 114, 112, 32, 68,
]

even = [((x ^ 66) - 4) ^ 55 for x in enc1]
odd = [((x ^ 0x13 ^ 0x4B) + 4) ^ 77 for x in enc2]

body = []
for a, b in zip(even, odd):
    body.extend((chr(a), chr(b)))

flag = "miniLctf{" + "".join(body) + "}"
print(flag)
assert len(flag) == 72
```

输出为：

```text
miniLctf{y0u_a1r4ady_und4rstand_th4_w1nd0ws_exc4pt1On_handl1e_m4chan1sm}
```

程序中“异常成功后修改 EIP”的跳转也解释了为何静态主流程看似总会落到失败分支：处理器通过改写上下文绕过失败路径，真正的控制流需要连同异常语义一起还原。

## 方法总结

Windows 逆向遇到无条件异常时，应检查 PE 的 TLS 目录、VEH 注册和 `__try/__except` 过滤器，而不是把异常一概当成反调试噪声。这里偶、奇字符被拆进两套处理器，还额外使用异常点的 EIP 低字节作为常量；把每个等式独立逆运算，再按索引交错，是最直接且可验证的恢复方法。
