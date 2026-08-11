# rocky

## 题目简述

程序接收最多 16 个字符，先计算 MD5 并与内嵌的 16 字节 digest 比较；只有匹配时，才把原输入作为 AES-128-CBC key、其倒序字符串作为 IV，解密内嵌密文输出 flag。题目提示强调应把比较常量当作 bytes 而不是直接当文本处理，并在得到 digest 后关注其调用的加密库。

因此这是“逆向提取校验数据 → 密码库命中 → 按程序语义解密”的链条，不是对 AES 或 MD5 进行密码学破坏。

## 解题过程

### 提取并破解 MD5 校验

从 `main` 的 `memcmp(hash, result, 16)` 提取 digest：

```text
70 92 4d 0c f6 69 f9 d2 3c ca bd 56 12 02 35 1f
```

将这 16 个原始字节按 MD5 digest 表示后用常见口令库比对，得到：

```text
emergencycall911
```

该字符串恰为 16 字节，也解释了程序为何使用 `char user_input[17]`：末尾一字节留给 NUL，而有效输入刚好是 AES-128 key 长度。

### 复现程序的 AES 参数

成功分支并不是直接打印输入，而是执行：

```c
reverse_string(user_input, rev_user_input);
AES_init_ctx_iv(&ctx, user_input, rev_user_input);
AES_CBC_decrypt_buffer(&ctx, decrypted, ciphertext_len);
```

所以 key 是 `emergencycall911`，IV 是其反转后的 `119llacycnegreme`。以这组参数对 `precomputed` 的全部字节做 AES-128-CBC 解密即可恢复 flag 文本。源目录的 `create_flag.py` 使用相同 key/IV 生成密文，是对逆向结论的静态验证。

### 验证

官方题解描述了“提取 hash、破解、再让程序执行 AES 解密”的同一流程；题目源码和生成脚本又确认了 MD5、倒序 IV 与 AES-CBC 三者的连接。本文未运行样本或口令库。

## 方法总结

- 核心技巧：把内嵌散列从反汇编常量恢复为字节 digest，再按成功分支的 key/IV 派生规则复现解密。
- 识别信号：输入首先经过标准 hash 比较，成功后被送入对称加密 API 时，散列破解通常只是取得解密密钥的前置步骤。
- 复用要点：不要把 digest 直接按可打印字符串抄出；同时记录 key 长度、IV 来源、模式和密文长度，才能与程序输出一致。
