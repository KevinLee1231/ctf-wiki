# Keys

## 题目简述

二进制内硬编码了一个 4 字节异或密钥，并用它循环处理 flag。没有复杂密钥派生或密码算法；从反编译中的循环和常量即可恢复明文。

## 解题过程

循环等价于：

$$
m_i=c_i\oplus k_{i\bmod4}.
$$

从数据段和异或指令提取密钥：

~~~text
4a 83 04 d5
~~~

再对 enc_flag 循环异或：

~~~python
key = bytes.fromhex("4a 83 04 d5")
plain = bytes(
    value ^ key[index % len(key)]
    for index, value in enumerate(enc_flag)
)
print(plain)
~~~

输出：

~~~text
maple{th3_KEY_IS_H4RDC0D3d_1n_THE_B1n4ry}
~~~

## 方法总结

硬编码秘密无法通过混淆循环获得真正保护。逆向时先抽象循环的输入、状态和周期；若状态只按固定小周期重复，通常可以直接写几行代码复现。恢复后应重新异或一次验证能回到原密文，避免抄错常量或端序。
