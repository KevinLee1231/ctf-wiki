# d3sky

## 题目简述

题目是带反调试和自定义 VM 的逆向。程序通过 TLS 中的 `IsDebuggerPresent()` 与异常类型决定 key 内容，用 RC4 解密 opcode，每次执行 3 条指令后再加密回去。VM 的有效校验是四字节滑动异或关系，恢复该关系后可直接反推 flag 或用 z3 求解。

## 解题过程

d3sky 的主逻辑是用 and-not 指令构建一个虚拟机，其他部分比较常规，主要如下：

TLS 函数中存在反调试逻辑，会使用

- `IsDebuggerPresent()` 的返回值决定后续行为。

- 如果处于调试状态，则触发地址访问异常；否则触发除零异常。

- 在 try 块中，把 `key[5]` 改成字符 `1`，也就是把 key `YunZhiJun` 改成 `YunZh1Jun`。

- 根据触发的异常给 key 追加字符串；正确路径应是除零异常，完整 key 为 `YunZhiJunAlkaid`。

- 使用这个 key 初始化 RC4 的 s box，解密最后一段长度为 74 的 opcode。进入虚拟机后，每次解密 3 条 opcode，执行后再加密回去。

理解这个思路后，解题其实很简单：插桩并记录日志观察逻辑即可。注意 opcode 解密后必须再加密回去。

虚拟机核心逻辑是每次把输入的 4 个字节异或，再与密文异或。如果结果为 0，就表示校验成功。写成代码就是下面的形式：

```
flag = b'A_Sin91e_InS7rUcti0N_ViRTua1_M4chin3~'
enc = []
for i in range(37):
  enc.append(flag[i]^flag[(i+1)%37]^flag[(i+2)%37]^flag[(i+3)%37])
```

看起来很多人使用 z3 求解；下面给出直接反推算法的脚本：

```
enc = [36, 11, 109, 15, 3, 50, 66, 29, 43, 67, 120, 67, 115, 48, 43, 78, 99, 72,
119, 46, 50, 57, 26, 18, 113, 122, 66, 23, 69, 114, 86, 12, 92, 74, 98, 83, 51]
dec = [0] * 37
dec[-1] = 126
for i in range(32, -1, -4):
  dec[i] = enc[i] ^ enc[i+1] ^ dec[i+4]
dec[33] = enc[33] ^ enc[34] ^ dec[0]
for i in range(29, 0, -4):
  dec[i] = enc[i] ^ enc[i+1] ^ dec[i+4]
dec[2] = enc[-1] ^ dec[0] ^ dec[1] ^ dec[-1]
for i in range(3, 37):
  dec[i] = enc[i-3] ^ dec[i-1] ^ dec[i-2] ^ dec[i-3]
print(bytes(dec))
```

## 方法总结

- 核心技巧：TLS 反调试、异常分支决定密钥、RC4 动态解密 opcode、自定义 VM 关系式还原。
- 识别信号：opcode 运行时解密又回写、调试状态改变异常路径时，应先还原正确运行环境下的 key，再插桩记录 VM 明文指令。
- 复用要点：一旦 VM 校验被化简成 `flag[i] ^ flag[i+1] ^ flag[i+2] ^ flag[i+3]` 这类线性关系，就不需要完整模拟所有控制流，直接反推或约束求解更稳。
