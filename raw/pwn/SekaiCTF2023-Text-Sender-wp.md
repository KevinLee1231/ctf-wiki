# Text Sender

## 题目简述

程序维护至多 10 条消息。每条消息由三个堆块组成：一个 0x20 大小的 `MESSAGE` 结构、一个申请大小为 `0x78` 的收件人字符串，以及一个申请大小为 `0x1f8` 的正文字符串。菜单允许设置发送者、添加、编辑、打印和发送消息；“发送”会依次释放这些堆块。

题目使用 glibc 2.32。利用链结合了越界比较泄露、`scanf("%Ns")` 产生的 off-by-null、House of Einherjar、Safe-Linking 下的 tcache poisoning，以及 `__free_hook` 劫持。

## 解题过程

### 1. 把前缀匹配变成逐字节泄露

`edit_message` 用 `getline` 读取任意长度的名字，却只按攻击者输入的长度与 `msg->receiver` 比较：

```c
for (ssize_t j = 0; j < length - 1 && check; j++) {
    if (name[j] != msg->receiver[j])
        check = 0;
}
```

代码既不要求两边长度相等，也不在收件人字符串结尾停止。短输入可以命中任意收件人的前缀；长输入则继续比较收件人堆块之后的用户数据和堆元数据。程序打印 `Old message` 代表全部猜测字节正确，否则打印 `Cannot find name`，由此形成一个字节相等性 oracle。

先反复添加、发送消息填充 tcache，再重新申请，使某个内容已知的 `receiver` 后方紧邻已释放块中的指针。对下一个未知字节枚举 $0$ 至 $255$，出现成功提示时固定该字节。官方脚本依次恢复：

- 一个堆指针，据此得到堆基址；
- unsorted bin 中指向 `main_arena` 的指针，并用题目所给 libc 的偏移 `0x1c5c10` 求出 libc 基址。

简化后的 oracle 形式如下：

```python
known = bytearray(prefix)
for _ in range(pointer_length):
    for guess in range(256):
        edit_name = bytes(known) + p8(guess)
        io.sendlineafter(b"Name: ", edit_name)
        if b"Old message" in io.recvline():
            known.append(guess)
            io.sendlineafter(b"New message: ", b"keep")
            break
```

实际利用中的 `prefix` 包含已知字符串、填充和相邻 chunk 的 `size` 字段，必须按现场堆布局构造。

### 2. 利用 off-by-null 触发 House of Einherjar

`input` 动态构造 `%Ns%*c`。当目标缓冲区恰好读入 $N$ 个非空白字节时，`scanf` 还会追加一个 NUL，因此向申请大小为 `0x78` 的 `receiver` 输入 0x78 字节，会把紧邻 `text` chunk 的 `size` 低字节从 `0x01` 清成 `0x00`：

```text
0x201  ->  0x200
```

这同时清除了 `PREV_IN_USE` 位。`receiver` 可用区末尾又与下一块的 `prev_size` 位置重叠，所以攻击者还可以写入伪造的回退距离。

[House of Einherjar](https://heap-exploitation.dhavalkapil.com/attacks/house_of_einherjar) 的必要条件在本题中全部满足：用单字节 NUL 清除下一块的 `PREV_IN_USE`，把其 `prev_size` 指向攻击者布置的假空闲块，并令假块的 `fd`、`bk` 指回自身，通过 `P->fd->bk == P` 与 `P->bk->fd == P` 检查。随后释放受害 `text`，glibc 会按伪造的 `prev_size` 向后合并，产生覆盖既有已分配区域的大空闲块。

官方脚本布置假块时使用的结构为：

```python
fake = flat(
    0,          # prev_size
    0x2850,     # fake chunk size / backward distance
    fake_addr,  # fd
    fake_addr,  # bk
)
```

之后再以精确的 0x78 字节 `receiver` 输入写入 `prev_size=0x2850`，由自动追加的 NUL 清除后继块的 P 位，最后通过“发送”释放并触发向后合并。

### 3. Safe-Linking tcache poisoning

重叠块使后续 `getline` 分配得到的区域能够覆盖 tcache 单链表指针。glibc 2.32 对 `next` 使用 Safe-Linking，写入值不是裸目标地址，而是：

$$
\text{encoded\_next} =
\text{target}\oplus\left(\text{chunk\_address}\gg12\right).
$$

将 `target` 设为 `__free_hook`，再执行两次对应大小的分配：第一次取走当前 tcache 项，第二次便返回 `__free_hook`。向其中写入 `system` 地址，同时准备一个内容为 `/bin/sh\x00` 的消息正文。

最后选择“发送”，程序执行：

```c
free(msg->text);
```

等价于调用 `system("/bin/sh")`，取得 shell。仓库中给出的 flag 为：

```text
SEKAI{y0U_Kn@W_h0W_tO_c@NduCt_H0uS3_@f_31Nh3rJ4r_43422bb9c023c5a8c37388316956e7c4}
```

## 方法总结

这条利用链的前半段不是传统打印泄露，而是越界比较形成的布尔 oracle；后半段则把 `%Ns` 的终止 NUL 转化为堆块标志位破坏。House of Einherjar 负责制造重叠，Safe-Linking 公式负责把重叠写升级为对 `__free_hook` 的定向分配。利用高度依赖题目附带的 glibc 2.32、堆块申请顺序和精确地址偏移，迁移到其他版本时不能照搬常数。
