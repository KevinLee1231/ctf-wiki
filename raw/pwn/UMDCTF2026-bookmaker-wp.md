# bookmaker

## 题目简述

题目把 QuickJS 嵌入原生程序，并向 JavaScript 暴露 `Ledger` 与 `Wire` 对象。`Ledger.view()` 返回直接别名到原生堆缓冲区的 `ArrayBuffer`，但 `Ledger.recycle()` 会释放该缓冲区，旧的 JavaScript 视图仍可继续读写，形成 use-after-free。

目标是利用相同尺寸对象重占该堆块，泄露 PIE 并改写函数指针。

## 解题过程

原生 `Wire` 结构大小为 `0x30`，关键字段布局为：

```text
0x00  dst
0x08  len
0x10  stamp
0x18  emit
0x20  quota
0x28  armed
```

先创建大小为 `0x30` 的 `Ledger`，保存其视图后释放：

```javascript
let ledger = new Ledger(0x30);
let stale = ledger.view();
ledger.recycle();
let wire = mintWire();
```

紧接着分配的 `Wire` 会重用同一个 tcache 块，因此 `stale` 现在别名到 `Wire`。从偏移 `0x18` 读取 `emit` 可泄露 `reject_dispatch` 的运行时地址；减去本地 ELF 中该符号的偏移即可得到 PIE 基址，再计算 `win` 和全局 `dispatch_hook` 地址。

接着通过旧视图把 `Wire` 改造成任意地址写原语：

```text
dst = &dispatch_hook
len = 8
```

调用 `wireWrite` 写入 `win` 的地址，随后 `settle()` 会经由 `dispatch_hook` 间接调用 `win`，打开 shell。读取 flag 得到：

```text
UMDCTF{tahmid-will-surely-finish_his_challenge_on_time!!}
```

## 方法总结

跨语言绑定中的所有权错误是本题核心。JavaScript 的 `ArrayBuffer` 生命周期没有与原生 `free` 同步，使一个普通视图变成稳定的 UAF 读写入口。选择与目标结构完全相同的 `0x30` 尺寸可以提高重占确定性，随后按结构偏移先泄露、再改写函数指针。
