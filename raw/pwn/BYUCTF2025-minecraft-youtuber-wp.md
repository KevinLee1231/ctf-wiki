# Minecraft YouTuber

## 题目简述

程序在堆上使用两个同为 24 字节的结构体：

```c
typedef struct { long uid; char username[8]; long keycard; } user_t;
typedef struct { long mfg_date; char first[8]; char last[8]; } nametag_t;
```

注册用户时只初始化 `uid` 和 `username`，没有初始化偏移 16 的 `keycard`。隐藏菜单项 7 会在 `keycard == 0x1337` 时输出 flag，因此决定性漏洞是同尺寸堆块复用后的未初始化字段。

## 解题过程

先注册普通用户，反复选择“收集装备”，直到随机得到 Name Tag。Name Tag 的 `last` 字段与未来 `user_t.keycard` 都位于块内偏移 16，因此把姓写成 `p64(0x1337)`：

```python
while b"Tag" not in line:
    send_menu(3)
    line = recvline()

sendline(b"name")
sendline(p64(0x1337))
```

选择“Change characters”会先释放当前用户，再释放 Name Tag，然后马上分配新的 24 字节用户块。glibc tcache 按后进先出返回刚释放的 Name Tag 块。新注册流程覆盖偏移 0 的 `uid` 和偏移 8 的用户名，却保留偏移 16 的旧 `last` 内容，于是新用户的未初始化 `keycard` 为 `0x1337`。

最后直接提交未显示在菜单说明中的选项 7，校验通过并得到：

```text
byuctf{th3_3xpl01t_n4m3_1s_l1t3r4lly_gr00m1ng}
```

## 方法总结

- 核心技巧：用同尺寸对象进行堆风水，使已控字段残留到新对象的未初始化权限字段。
- 识别信号：不同结构体尺寸相同、生命周期相邻、构造函数未覆盖所有字段时，应逐偏移比较复用后的数据布局。
- 复用要点：利用依赖分配器 bin 大小和释放顺序；先确认块确实进入同一 tcache bin，并注意后进先出的复用规律。
