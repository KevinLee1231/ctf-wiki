# N1CTF 2021 - funny_misc

## 题目简述

附件是一块包含两个分区的硬盘镜像。第一个分区未加密，保存 GRUB 配置、Linux 内核和一个无法直接识别的 `encrypted_initrd.bin`；第二个分区由 LUKS 加密。启动链必须先解密 initramfs，再由其中的密钥文件打开根分区。

虽然载体是磁盘镜像，真正阻断后续取证的是内核中的 initramfs 解密逻辑，因此本文按决定性障碍归入 Reverse。

## 解题过程

### 从启动配置定位信任链

第一个分区的 `grub.cfg` 给出了实际启动参数：

```text
menuentry "Buildroot" {
    linux /boot/bzImage root=/dev/sda2 rootwait console=tty1
    initrd /boot/encrypted_initrd.bin
}
```

`file encrypted_initrd.bin` 只返回 `data`，说明它不是普通的 gzip/cpio initramfs。由于内核必须在挂载 `/dev/sda2` 前解开它，解密代码和密钥只能位于 `bzImage` 的早期启动路径中。

### 逆向内核中的循环 XOR

在解压后的内核中追踪 initrd 缓冲区的读取与写入，可以找到一个逐字节处理循环。原 WP 中唯一的反编译截图只是下列代码的视觉形式，没有额外图形信息，因此直接转写：

```c
const char *key = "ThI1s_IssSs_KkeEeyy";
size_t key_len = strlen(key);

for (size_t i = 0; i < initrd_len; i++) {
    initrd[i] ^= key[i % key_len];
}
```

写一个流式脚本即可恢复原始 initramfs，无需一次性申请固定的 50 MiB 缓冲区：

```python
key = b"ThI1s_IssSs_KkeEeyy"

with open("encrypted_initrd.bin", "rb") as src, \
     open("decrypted_initrd.bin", "wb") as dst:
    offset = 0
    while chunk := src.read(1 << 20):
        dst.write(bytes(b ^ key[(offset + i) % len(key)] for i, b in enumerate(chunk)))
        offset += len(chunk)
```

重新执行 `file`，再按识别出的压缩格式解压并展开 cpio，即可得到完整 initramfs。关键文件包括 `/init` 与 `/keyfile`。

### 用 initramfs 密钥打开第二分区

`/init` 已经把真实启动流程写得很清楚：

```sh
cryptsetup luksOpen --key-file /keyfile /dev/sda2 rootfs
mount /dev/mapper/rootfs /mnt/rootfs
exec switch_root -c /dev/console /mnt/rootfs /sbin/init
```

离线分析时，为原始磁盘镜像建立带分区扫描的 loop 设备，并复用同一密钥：

```sh
loopdev=$(losetup --find --show --partscan disk.img)
cryptsetup luksOpen --key-file ./keyfile "${loopdev}p2" n1ctf-root
mkdir -p ./rootfs
mount /dev/mapper/n1ctf-root ./rootfs
```

挂载后的根目录中可以看到 `flag.png`。仓库只保留了官方文字 WP 与内核反编译截图，没有提供原始磁盘、解密后的 initramfs 或最终 `flag.png`，所以无法在当前材料中转录具体 flag。

## 方法总结

分析加密启动介质时应按启动依赖逆向追踪：GRUB 指向内核和 initramfs，内核必须掌握第一层解密逻辑，initramfs 又必须携带打开 LUKS 根分区的密钥。只要沿这条信任链逐层验证文件格式和挂载结果，就不需要暴力破解 LUKS。纯代码截图应转写成可搜索、可复制的文本，而不作为独立图片保留。
