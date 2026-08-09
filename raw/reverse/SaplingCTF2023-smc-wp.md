# smc

## 题目简述

程序启动后调用 mprotect 把代码页改为可写可执行，并用 32 字节循环密钥异或解密 prog_start 到 prog_end，随后恢复 RX 权限。这是自修改代码；静态反编译磁盘上的加密区域会得到错误结果。

## 解题过程

可在 init 返回后下断点并转储已解密的代码页，也可从源码提取密钥手工异或。恢复出的 main 把 36 字节输入视为 9 个小端 32 位整数，执行：

~~~c
for (i = 0; i < 8; i++)
    input[i] = (input[i] ^ input[i + 1]) + k[i];
input[8] += k[8];
~~~

从末尾向前逆推。设目标数组为 y：

~~~python
x[8] = (y[8] - k[8]) & 0xffffffff
for i in range(7, -1, -1):
    x[i] = ((y[i] - k[i]) & 0xffffffff) ^ x[i + 1]
flag = b"".join(v.to_bytes(4, "little") for v in x)
~~~

结果为：

~~~text
maple{1s_Th1s_53lf_Modify1ng_C0d3??}
~~~

## 方法总结

遇到 mprotect 后写代码段，应优先考虑运行时解密。转储点必须在解密完成后、代码再次被修改前。算法层面要注意 C 的 uint32 环绕和小端打包；递推含相邻输入时通常应从已知边界反向求解。
