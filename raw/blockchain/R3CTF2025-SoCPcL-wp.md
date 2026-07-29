# SoCPcL

## 题目简述

题目提供一份打过补丁的 Solana validator。补丁在 `solana-test-validator` 进程中用 `lazy_static` 读取 `./flag`，因此 flag 只存在于验证器宿主进程的内存里，链上程序没有正常接口可以取得它。

利用目标不是攻击某个业务合约，而是借助 Solana BPF（SBF）程序、跨程序调用和账户内存映射之间的实现细节，把一个账户数据切片变成宿主地址空间的任意读原语，再定位并读出保存 flag 的 Rust `String`。

## 解题过程

### 制造便于辨认的账户映射

利用程序接收五个账户，依次作为 `receiver`、`payer`、`victim`、PDA 和 `callee`。客户端为 `receiver` 分配 `0x100000` 字节，为另外两个可写账户分配较小空间；程序先把 `victim` 的开头填成 `0x3e`，作为布局失败时的显眼标记。

随后程序通过 `invoke_signed()` 调用一个几乎为空的 callee。跨程序调用会促使运行时建立、切换账户的直接映射区域，也使相关 `MemoryRegion` 元数据出现在当前 SBF 实例能够扫描到的宿主内存附近。

### 从 `AccountInfo` 找到映射元数据

`AccountInfo.data` 的实际类型是：

```rust
Rc<RefCell<&mut [u8]>>
```

官方利用代码在 `unsafe` 中取得 `Rc` 内部地址，再取出切片保存的宿主数据指针。它从 1 MiB 的 `receiver` 数据附近扫描如下特征：

```text
vm_addr = 0x0000000400000060
vm_end  = 0x0000000400100060
vm_len  = 0x100000
```

同时还检查相邻位置的固定边界值，避免仅凭三个数字误命中。找到 `receiver` 的 `MemoryRegion` 后，利用固定的相邻布局定位 `victim` 的区域描述符。描述符的首字段是映射对应的宿主地址，而 `victim.data` 中缓存的数据指针仍指向同一虚拟映射入口。

### 把账户读取改造成宿主任意读

关键原语只需临时替换 `victim` 区域描述符的宿主基址：

```rust
fn read(region: *mut u64, victim_data: *mut u64, addr: u64) -> u64 {
    unsafe {
        let original = *region;
        *region = addr;
        let value = *victim_data;
        *region = original;
        value
    }
}
```

从 SBF 程序看来，它仍在读取合法的账户数据；实际映射目标却已被改成任意宿主地址。每次读取后恢复原字段，可以继续稳定使用该账户。

### 定位 flag 并带回链上

利用代码根据区域描述符附近的固定代码指针减去已知偏移，算出 validator 可执行文件基址；再读取一个 GOT 项并换算 libc 基址，用高位特征判断本轮随机布局是否符合预期。官方 PoC 对这一步采用版本相关的固定偏移，因此客户端会重复创建实例，直到命中可用布局。

补丁生成的 `FLAG` 最终对应一个全局 Rust `String`。在已知可执行文件基址后，PoC 计算其对象地址，依次读取：

```text
String.len
String.ptr
String.ptr 指向的 len 个字节
```

得到 flag 后，程序把字节复制进 `payer` 账户数据。客户端交易完成后读取该账户；若开头仍是 `0x3e3e3e3e` 就重试，否则输出返回的数据。

## 方法总结

本题的决定性障碍是 Solana 运行时的账户直接映射，而不是普通的 Rust 越界。利用链可以概括为：用 CPI 塑造账户映射布局，从 `AccountInfo.data` 泄露映射入口，扫描 `MemoryRegion` 元数据，篡改区域宿主基址获得任意读，最后按 Rust `String` 布局读取 validator 进程中的全局 flag。

官方 `caller.rs`、`callee.rs` 和 `poc.js` 已给出完整原语与重试逻辑；其中地址偏移依赖题目附带的 validator 构建，不能直接当作通用 Solana 利用参数。
