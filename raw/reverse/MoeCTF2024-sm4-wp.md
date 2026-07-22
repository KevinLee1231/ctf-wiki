# sm4

## 题目简述

程序使用标准 SM4-ECB 对固定 48 字节输入加密，并与 48 字节密文比较。真正的陷阱不是 SM4 轮函数，而是无宽度限制的 `scanf("%s", Data_plain)`：正确 flag 恰好 48 字节，结尾 NUL 会越界写到相邻 `key[0]`，把实际密钥首字节从 `t` 改为 `0x00`。

## 解题过程

源码中看似定义了：

```c
unsigned char key[16] = "thekeytosomethin";
unsigned char Data_plain[48] = {0};
scanf("%s", Data_plain);
encode_fun(48, key, Data_plain, encode_Result);
```

但反汇编确认该构建中 `Data_plain` 后面紧邻 `key`。`scanf` 写入 48 个可见字符后还会写一个字符串终止符，于是运行时真正使用的 key 为：

```text
00 68 65 6b 65 79 74 6f 73 6f 6d 65 74 68 69 6e
```

也就是十六进制 `0068656b6579746f736f6d657468696e`。目标密文为：

```text
ad6ccdc109fcddef83ae9308538ec537
5cdd1b4b039919a26924964277c1275f
2dd45df52bb032f7a597c68aee48ae93
```

SM4 分组为 16 字节，三块均无填充。可直接用支持 SM4 的 OpenSSL 复现解密，无需把整份 S-box 和轮函数复制进 WP：

```bash
printf '%s' \
  'ad6ccdc109fcddef83ae9308538ec5375cdd1b4b039919a26924964277c1275f2dd45df52bb032f7a597c68aee48ae93' \
  | xxd -r -p \
  | openssl enc -d -sm4-ecb \
      -K 0068656b6579746f736f6d657468696e \
      -nopad
```

输出：

```text
moectf{Congratulations_you_are_an_SM4_master!!!}
```

该字符串长度正好为 48；把它输入程序时，越界 NUL 会再次生成同一把有效密钥，因而加密结果与内置密文一致。

## 方法总结

标准密码算法题也要检查调用点的内存语义。源码中声明的 key 不一定是运行时 key，数组相邻关系也必须以目标二进制为准；这里一个 NUL off-by-one 改变了 SM4 密钥首字节，解释了使用表面密钥始终解不出的现象。
