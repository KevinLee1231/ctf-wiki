# 看不见的彼方：交换空间

## 题目简述

判题器接收 `选项@Alice 程序的 Base64@Bob 程序的 Base64`，把两个程序分别放进 `/home/pwn/A/space/exe` 与 `/home/pwn/B/space/exe`，再以 `pwn` 用户并发执行两个独立的 `chroot`。两个根文件系统只读，双方仅能写各自的 `/space`，但仍共享回环网络。判题限时 10 秒、容器内存 316 MiB、PID 总数 32；两个 `/space` 都是占用实际内存的 tmpfs。

第一问要求交换两个 128 MiB 文件。第二问要求把 Alice 的 128 MiB 文件送到 Bob，同时把 Bob 的两个 64 MiB 文件送到 Alice。限制的关键在于不能再完整复制一份数据，否则 tmpfs 页、进程和运行时开销会超过内存上限。

这不是云身份、Kubernetes、控制面或内存破坏问题，Docker 只是部署环境。决定性能力是在受限 Linux 运行时中用分块通信和文件打洞维持低峰值内存；现有正式安全方向都不能稳定覆盖这种资源约束系统编程题，因此暂归 `_unclassified`。

## 解题过程

### 第一问：分块原地交换

让 Alice 监听 `127.0.0.1:8080/exchange`，Bob 作为客户端。双方每次只处理 1 MiB，循环 128 次：

1. Alice 从自己文件的当前偏移读取块 `A[i]` 到小缓冲区。
2. Bob 读取 `B[i]`，在 POST 请求体中发送给 Alice。
3. Alice 用 `B[i]` 覆盖刚才读过的原位置，并把 `A[i]` 作为响应体返回。
4. Bob 收到响应后回退文件偏移，用 `A[i]` 覆盖 `B[i]`。

服务端的核心顺序可以压缩为：

```go
old := make([]byte, 1<<20)
io.ReadFull(file, old)
file.Seek(-int64(len(old)), io.SeekCurrent)

incoming := make([]byte, 1<<20)
io.ReadFull(requestBody, incoming)
file.Write(incoming)
response.Write(old)
```

这样任何时刻只多出若干 1 MiB 缓冲区，不会再分配 128 MiB 的第三份文件。

### 第二问：传一块，释放一块

第二问不能直接沿用上面的覆盖方式，因为源文件和目标文件的拆分关系不同，必须在增长新文件的同时释放旧文件占用的 tmpfs 页。Linux 的 `fallocate(2)` 支持对文件区间打洞：

```go
const (
    keepSize  = 0x01 // FALLOC_FL_KEEP_SIZE
    punchHole = 0x02 // FALLOC_FL_PUNCH_HOLE
)

func releaseCurrentChunk(f *os.File, n int64) error {
    if _, err := f.Seek(-n, io.SeekCurrent); err != nil {
        return err
    }
    off, err := f.Seek(0, io.SeekCurrent)
    if err != nil {
        return err
    }
    if err := syscall.Fallocate(int(f.Fd()), keepSize|punchHole, off, n); err != nil {
        return err
    }
    _, err = f.Seek(n, io.SeekCurrent)
    return err
}
```

`KEEP_SIZE | PUNCH_HOLE` 保持文件逻辑长度不变，却把该区间变成读回全零的洞并释放其物理页。调用时必须先把当前 1 MiB 读入缓冲区，再打洞；否则原数据已经丢失。

双方先创建目标文件并 `Truncate` 到最终长度。对 tmpfs 而言这只是建立稀疏逻辑长度，不会立即为所有数据页分配内存。随后仍使用逐块 HTTP 交换：

- Alice 每轮从 128 MiB 的 `file` 读一块、对刚读区间打洞、把 Bob 发来的块依次写入 `file1` 和 `file2`，同时把自己的旧块作为响应。
- Bob 先处理 `file1` 的 64 块，再处理 `file2` 的 64 块；每读一块就给源区间打洞，并把 Alice 返回的块顺序写入新建的 `file`。

因此每写入一块目标数据，几乎同步释放一块源数据，文件总逻辑大小虽未降低，实际驻留的 tmpfs 数据页总量保持近似不变。

### 运行时约束与验证

官方实现用 Go 时显式设置：

```go
runtime.GOMAXPROCS(2)
debug.SetMemoryLimit(10 << 20)
```

还需剥离符号并压缩二进制，以满足上传体积；这不是算法核心，但能避免 Go 运行时线程、堆和可执行文件副本吃掉余量。

判题器在两个进程结束后检查目标都是普通文件，并以交换前后的 SHA-256 对应关系验证全部内容。成功条件不是“目标看起来大小正确”，而是三份目标文件逐一与各自源哈希完全一致。

## 方法总结

- 核心技巧：固定大小的双向分块交换解决等长原地置换；`fallocate(FALLOC_FL_KEEP_SIZE | FALLOC_FL_PUNCH_HOLE)` 在拆分/合并时及时释放源文件物理页。
- 识别信号：tmpfs、严格内存上限、只读 rootfs、两个隔离目录但允许本机通信，以及“新文件增长会导致 OOM”，共同指向稀疏文件和逐块释放。
- 复用要点：区分文件逻辑长度与实际占用；严格遵守“读入缓冲区 -> 打洞 -> 发送/写入”的顺序，并把语言运行时、线程数和上传二进制体积计入资源预算。
