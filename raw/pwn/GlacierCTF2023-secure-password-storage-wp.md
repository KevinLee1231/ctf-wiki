# GlacierCTF2023 - Secure Password Storage

## 题目简述

程序把密码条目存入堆上的单链表。条目依次包含 1 字节哈希类型、8 字节轮数、8 字节 next 指针和 16 至 64 字节摘要。用户可新增、查看、重哈希和删除条目；目标二进制启用完整保护，并动态使用 OpenSSL 3。

## 解题过程

漏洞链从三个实现错误开始：

- salt 由用户提供的 8 字节与 `/dev/urandom` 做 AND；输入全零即可把 salt 固定为零；
- 查找条目时把摘要强转为 `short`，只比较最前两个字节；
- `rehash` 用用户新提交的哈希类型决定写入长度，而不按原 chunk 的分配尺寸写入。

因此可离线搜索一个密码，使 `MD5(password || zero_salt)` 与 `SHA256(...)` 的前两字节相同。先以 MD5 创建 16 字节摘要 chunk，再要求按 SHA-256 重哈希；查找仍命中原条目，却向 16 字节区域写入 32 字节摘要，覆盖相邻 chunk。

第一阶段令溢出最后一字节把下一条目的哈希类型从 MD5 改成 SHA-512。`view` 会按被篡改的类型多打印 48 字节，从相邻的已释放 `getline`/OpenSSL chunk 中获得堆指针和 libcrypto/libc 指针。

第二阶段利用相同越界写覆盖 tcache next 指针的最低字节。需要实时搜索一个同时满足以下条件的密码：MD5/SHA-256 前两字节碰撞，且重哈希结果最后一字节等于目标 safe-linking 指针的低字节。官方 C 辅助程序枚举密码并检查：

```c
if (*(uint16_t *)hash_md5 == *(uint16_t *)hash_sha256 &&
    hash_sha256_next[31] == wanted_byte) {
    printf("PWD: %s\n", pwd);
    exit(0);
}
```

将 tcache 链指向相邻的 OpenSSL 对象后，用一次 `getline` 分配覆盖它。伪造对象内的回调指针为 `system`，作为参数使用的字段指向 `/bin/sh`；随后程序在查看条目流程中调用该回调并弹出 shell，读取：

```text
gctf{th3_best_typ3s_0f_ch4llenges_do_be_h3ap_fuck3ry}
```

## 方法总结

低位摘要比较把跨算法条目混为一谈，可预测 salt 又把碰撞搜索变成完全离线；错误的重哈希尺寸最终提供堆越界写。评估密码存储代码时，算法标识、摘要长度、salt 和比较长度必须绑定为同一条目的不可分割元数据。
