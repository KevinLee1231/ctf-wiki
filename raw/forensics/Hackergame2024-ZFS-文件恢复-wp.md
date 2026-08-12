# ZFS 文件恢复

## 题目简述

附件是一个 ZFS 池镜像。`hg2024/data` 当前目录为空，但存在快照 `hg2024/data@mysnap`。生成脚本先打开 `flag1.txt` 和 `flag2.sh` 的文件描述符，再删除目录项、创建快照，最后才关闭描述符。ZFS 为保证崩溃一致性，会把这种“已 unlink 但仍被打开”的对象保存在 delete queue 中，因此普通挂载和目录遍历看不到文件，数据块与 ZFS 属性却仍可由 `zdb` 恢复。

## 解题过程

### 定位快照与 delete queue

建议只在镜像副本上操作，并以只读方式导入池。导入后列出 dataset 和快照：

```bash
loopdev=$(sudo losetup --find --show --partscan --read-only zfs.img)
sudo zpool import -o readonly=on -d "$loopdev" hg2024
zfs list -rt all hg2024
```

可以看到 `hg2024/data@mysnap`。ZPL 的 master node 固定为对象 1，使用最高详细度查看：

```bash
sudo zdb -ddddd hg2024/data@mysnap 1
```

关键输出为：

```text
DELETE_QUEUE = 33
ROOT = 34
```

继续查看对象 33：

```bash
sudo zdb -ddddd hg2024/data@mysnap 33
```

它是 `ZFS delete queue`，包含两个条目：

```text
2 = 2
3 = 3
```

这两个 dnode 就是被删除的两个文件。

### 恢复 flag1.txt

查看对象 2：

```bash
sudo zdb -ddddd hg2024/data@mysnap 2
```

输出表明它是大小 4135 字节、权限 `100644` 的普通文件，前 4096 字节位于普通 L0 block，剩余 39 字节被放进 embedded block pointer。读取第一个数据块：

```bash
sudo zdb -R hg2024 0:20e00:1000/a00:d
```

块末尾是 `fl`。embedded 数据不能再按普通磁盘偏移读取，需要先 dump L1 block，取出其中的 embedded block pointer，再交给 `zdb -E` 解码：

```bash
sudo zdb -R hg2024 0:21800:20000/400:d
sudo zdb -E 74302eaf4c4b9c78:cbcdcdcbf3f3ccf4:78928a8ad712e203:22bacae2c4d48353:2e5abafb128cf4c3:1828c1460a305186:8013008a90000fff:3051828c1460a305:60a3051828c1460a:0e341800005d0c14:000000000000000b:00000000000000b2:0000000000000000:0000000000000000:0000000000000000:0000000000000000
```

`zdb -E` 输出从 `ag{` 开始的剩余部分，与上一块末尾的 `fl` 拼接得到：

```text
flag{p1AInNNmmnnmmntExxt_50easy~r1ght?~}
```

这里容易误判为块损坏：题目特意设置 `recordsize=4k` 和 gzip 压缩，并用 4094 个随机小写字母让 `fl` 卡在第一块末尾，使 flag 剩余部分进入 embedded block pointer。

### 恢复 flag2.sh

对象 3 是可执行脚本。根据其 dnode 的数据块指针读取内容：

```bash
sudo zdb -ddddd hg2024/data@mysnap 3
sudo zdb -R hg2024 0:20800:200
```

还原出的关键脚本为：

```sh
#!/bin/sh
flag_key="hg2024_$(stat -c %X.%Y flag1.txt)_$(stat -c %X.%Y "$0")_zfs"
echo "46c518b175651d440771836987a4e7404f84b20a43cc18993ffba7a37106f508  -" > /tmp/sha256sum.txt
printf "%s" "$flag_key" | sha256sum --check /tmp/sha256sum.txt || exit 1
printf "flag{snapshot_%s}\n" "$(printf "%s" "$flag_key" | sha1sum | head -c 32)"
```

GNU `stat -c %X.%Y` 分别展开文件的 atime 与 mtime 的 Unix 时间戳。两个 dnode 的属性给出：

```text
object 2: atime=1141919810, mtime=233696969
object 3: atime=2109876543, mtime=1357924680
```

因此：

```text
flag_key=hg2024_1141919810.233696969_2109876543.1357924680_zfs
```

先用脚本中的 SHA-256 常量验证 `flag_key` 完全正确，再计算其 SHA-1 并取前 32 个十六进制字符，得到：

```text
flag{snapshot_6db0f20dd59a448d314cb9cabe8daea9}
```

恢复完成后可执行：

```bash
sudo zpool export hg2024
sudo losetup --detach "$loopdev"
```

`--show` 会把实际分配的 loop 设备记入 `loopdev`，因此导入和清理始终针对同一设备，不依赖它恰好是 `/dev/loop0`。

## 方法总结

- 核心技巧：从 ZFS 快照的 master node 找到 delete queue，再按 dnode、block pointer 和 embedded block pointer 逐层恢复文件内容与时间属性。
- 识别信号：目录为空但快照 `REFER`/`USED` 非零；文件在删除时仍有打开的 fd；ZFS master node 暴露 `DELETE_QUEUE` 对象号。
- 复用要点：`zdb -d` 的重复次数决定元数据详细度，`zdb -R` 读取普通块，`zdb -E` 解码 embedded block pointer。恢复脚本时不能只取正文，还要保留 atime、mtime、权限等元数据，因为它们可能直接参与密钥派生。
