# bi0sCTF 2024 - fullchain

## 题目简述

`fullchain` 把三道独立漏洞题串进同一隔离层级：先利用 `ezv8 revenge` 在 QEMU 客体中的 d8 进程取得普通用户代码执行，再利用 `palindromatic` 内核模块提升为客体 root，最后通过 `virtio-note` 设备的越界访问控制 QEMU 进程，在宿主侧读取 flag。

服务除 JavaScript 外还接收一个下载 URL，把攻击归档作为额外磁盘传入虚拟机。归档中应准备客体提权程序和 QEMU 逃逸程序；JavaScript 阶段获得 shell 后再挂载或复制这些文件，依次执行完整链。

## 解题过程

### 第一阶段：d8 用户态代码执行

V8 补丁仍是 `InferMapsUnsafe` 漏掉 `JSCreate` 副作用。使用 Proxy 作为 `Reflect.construct` 的 `newTarget`，在 getter 中把 holey double 数组改成对象元素，令优化代码继续使用过期 map。官方 `exp.js` 将该类型混淆扩展成长度为 `0x8000` 的 OOB 数组，再构造：

- 压缩地址 `addrof`；
- 通过篡改 double 数组 elements 指针实现的 cage 内 64 位读写；
- 两个 Wasm 实例，其中第一个用 `f64.const` 立即数携带 shellcode，第二个的代码指针被改到 shellcode。

调用第二个导出函数后执行 `execve("/bin/sh")`，获得客体中的普通用户 shell。所有数组下标、Wasm 字段偏移和代码内偏移都与附件 d8 构建绑定，应保留官方版本进行利用。

### 第二阶段：palindromatic 客体提权

普通用户接着运行归档中的 `lpe`。`SANITIZE` 对请求的处理存在单字节零溢出，可把相邻 request 的 `type` 改坏，使同一指针同时留在 incoming/outgoing 两个队列。通过 `RESET`、`PROCESS` 和 `REAP` 的特定顺序制造 UAF，并利用 cross-cache 让悬空 request 与 `pipe_buffer` 重叠。

官方利用随后：

1. 释放 victim `pipe_buffer`，用 `msg_msgseg` 占据同一对象；
2. 通过 System V 消息读取被 `splice` 更新的 `pipe_buffer`；
3. 伪造 `flags = PIPE_BUF_CAN_MERGE`、`offset = 0`、`len = 0`；
4. 让后续 pipe 写入覆盖只读打开的 `/etc/passwd` 页面缓存；
5. 写入已知 root 密码哈希并切换到 root。

这一阶段绕过的是客体 Linux 内核权限，尚未离开 QEMU。

### 第三阶段：virtio-note 逃逸

root 权限允许访问 `/dev/virtionote`。设备后端对正数 `idx` 未校验上界，可越界访问 QEMU 堆上的 note 指针数组。官方 `pwn` 先从固定 OOB 索引泄漏 `VirtIONote` 对象地址，然后把一个越界 note 指针改成指针数组自身，形成任意地址读写：

```c
vn_write(0x13, vnote + 0x210);  /* 让越界槽指向指针数组 */

void arb_write(unsigned long addr, unsigned long value) {
    vn_write(0x1e, addr);
    vn_write(0x0, value);
}
```

从 `vnote->vnq` 泄漏 virtqueue，从 `VirtIONote` 内的代码指针减去已知偏移得到 QEMU PIE 基址。在 QEMU 堆中写入 `flag.txt` 和 `open/read/write` ROP 链，再把 `vnote->vnq->handle_output` 改成栈迁移 gadget。最后发起一次设备请求触发回调，ROP 在宿主 QEMU 进程中打开并输出宿主工作目录的 flag。

整个层级关系是：

```text
d8 JIT 类型混淆
  -> 客体普通用户 RCE
  -> palindromatic UAF / Dirty Pipe 风格覆盖
  -> 客体 root
  -> virtio-note QEMU 堆任意读写
  -> 宿主 QEMU 进程 ORW
```

## 方法总结

这道题不能在获得第一个 shell 后停止：d8、客体内核和 QEMU 分别是三层独立边界。V8 漏洞提供初始执行，palindromatic 提供访问设备所需的客体权限，virtio-note 才真正读取宿主 flag。调试时应逐层设置成功标志和地址校验；任一层使用的偏移都必须与该层附件二进制、内核或 QEMU 构建匹配。
