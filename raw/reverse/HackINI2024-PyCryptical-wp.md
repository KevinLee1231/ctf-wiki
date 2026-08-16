# HackINI2024 PyCryptical

## 题目简述

服务每次连接都会生成一个 10 字符随机密钥，并输出密钥及加密后的 passphrase。附件是 Python 2.7 的 `.pyc` 字节码。需要反编译字节码恢复加密公式，解出本次连接的 passphrase，提交后取得 flag。

## 解题过程

先确认文件类型：

```bash
file chall.pyc
```

结果表明它是 Python 2.7 byte-compiled 文件。可使用固定路径下的 `pycdc` 反编译：

```bash
/home/kali/pycdc/build/pycdc chall.pyc > chall.py
```

恢复出的核心加密逻辑为：

```python
encrypted_char = (
    ord(plain_string[i]) * ord(key[i % len(key)])
    + pow(i, 16)
)
```

设第 $i$ 个密文整数为 $c_i$，循环密钥字符为 $k_i$，则明文字符码为：

$$
p_i=\frac{c_i-i^{16}}{\operatorname{ord}(k_i)}
$$

题目构造保证整除，可以直接实现：

```python
def decrypt(ciphertext, key):
    return "".join(
        chr((value - index ** 16) // ord(key[index % len(key)]))
        for index, value in enumerate(ciphertext)
    )

# key 和 ciphertext 使用当前连接实际输出的值
print(decrypt(ciphertext, key))
```

对仓库官方求解样例中的密钥 `jwpO4T6ktg` 和对应整数列表计算，得到：

```text
R3vers3_My_B3l0v3d__r3veR5e
```

该值也与容器构建脚本设置的 `PASSPHRASE` 一致。提交后服务输出：

```text
shellmates{Cr4ck3D_PyC_As_4_5harP_p1k3}
```

## 方法总结

这道题的主要障碍是从 Python 2 `.pyc` 恢复程序语义。定位逐字符公式后，乘法和加法都是可逆的；密钥虽然每次重连都会变化，但服务同时公开密钥和密文，所以求解器只需解析当前输出。复现时应使用同一次连接的一对数据，不能把旧密钥与新密文混用。
