# pwnymalloc

## 题目简述

程序使用自定义显式空闲链表分配器。退款对象包含 `status`、`amount` 和 0x80 字节理由，创建时状态始终被设为 `REFUND_DENIED`；只有把既有对象的状态改成 `REFUND_APPROVED` 才会打印 Flag。普通业务逻辑没有修改状态的入口，漏洞位于释放投诉对象时的向前合并。

## 解题过程

已用块只有 8 字节大小头；空闲块还把用户区开头作为双向链表指针，并在块尾保存 8 字节 boundary tag。`prev_chunk` 无条件把当前块前 8 字节解释成前块大小：

```c
return (chunk_ptr)((char *)block - get_prev_size(block));
```

但 boundary tag 只存在于空闲块。若物理前块仍在使用，这 8 字节实际属于用户可控数据。成熟分配器会用 `PREV_INUSE` 一类状态位先判断前块是否空闲，本实现没有该保护。

退款结构大小为 0x88，加 8 字节头并按 16 字节对齐后，每个退款块占 0x90。投诉申请 0x48 字节，加头后占 0x50。依次申请两个退款和一个投诉，堆布局就是 `refund0 | refund1 | complaint`。

第一份退款理由在偏移 0x68 处写入伪块头 `size=0x50`，其后 16 字节清零作为 `next/prev`。第二份退款理由末尾写入伪 footer `0xa8`。释放相邻投诉时，分配器从投诉头向前减 0xa8，恰好落到第一份退款内部的伪头；它看到低位为零，便把伪块当作空闲前块进行合并，并把这个横跨两份退款数据的区域插入空闲链表。

```python
from pwn import p32, p64

# fgets 最多接收 0x7f 字节，最后一字节由程序强制补零。
refund_req(0, b"\0" * 0x68 + p64(0x50) + b"\0" * 0x0F)
refund_req(0, b"\0" * 0x78 + p64(0xA8)[:-1])
complain(b"A")  # 立即清零并释放，触发使用伪 footer 的合并

# 下一次 0x90 退款申请复用重叠空闲块。
# 新对象 reason+0x10 正好覆盖原 request[1]->status。
refund_req(0, b"\0" * 0x10 + p32(1) + b"\n")
refund_status(1)
```

新对象自己的 `status` 最后仍会被程序写成 0，但写入理由时已把旧的第二份退款状态改成枚举值 1。检查 ID 1 后得到：

```text
uiuctf{the_memory_train_went_off_the_tracks}
```

## 方法总结

- 决定性缺陷是“当前块前方一定存在可信 footer”的错误假设；在前块仍被占用时，该字段完全由用户控制。
- 伪头 `0x50` 决定合并后记录的大小，伪 footer `0xa8` 决定向前跳转距离，两者在此实现中没有一致性校验，因此可以分别布置。
- 堆利用应逐字节计算结构、分配器头部、对齐和 `fgets` 的 0x7f 字节上限；最终重叠写为何命中 `request[1]->status` 必须能由偏移推导出来。
