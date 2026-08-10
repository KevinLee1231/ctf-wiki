# ez7621

## 题目简述

附件是友华 WR1200JS 路由器的 OpenWrt 固件 `openwrt-ramips-mt7621-youhua_wr1200js-squashfs-sysupgrade.bin`。目标是解包 SquashFS 根文件系统，定位随固件启动加载的 `mt7621-flag.ko` 内核模块，再从 MIPS 小端模块的 `init_module` 中恢复一个逐字节异或的字符串。

题目在官方 PDF 中放在 IOT 章节，但解题不依赖串口、JTAG、总线或其他物理硬件机制；决定性步骤是固件解包和 MIPS 内核模块逆向，因此归档到 Reverse。

## 解题过程

### 解包固件并定位模块

先让 `binwalk` 递归提取固件中的分区和文件系统：

```bash
binwalk -Me openwrt-ramips-mt7621-youhua_wr1200js-squashfs-sysupgrade.bin
```

进入提取出的 SquashFS 根目录并搜索带 `flag` 的文件：

```bash
cd _openwrt-ramips-mt7621-youhua_wr1200js-squashfs-sysupgrade.bin.extracted/squashfs-root
find . -iname '*flag*'
```

结果中既有 OpenWrt 的模块加载配置和包管理元数据，也有实际代码：

```text
./etc/modules.d/30-flag
./etc/modules-boot.d/30-flag
./lib/modules/5.15.137/mt7621-flag.ko
./usr/lib/opkg/info/kmod-flag.control
./usr/lib/opkg/info/kmod-flag.list
./usr/lib/opkg/info/kmod-flag.prerm
```

`/etc/modules-boot.d/30-flag` 表明该模块会在启动早期加载，但不必为了取 flag 真正启动固件或加载未知内核模块。直接检查 ELF 头即可确定分析参数：

```bash
file lib/modules/5.15.137/mt7621-flag.ko
readelf -h lib/modules/5.15.137/mt7621-flag.ko
```

MT7621 使用 MIPS 架构，固件中的 OpenWrt 目标为小端 `mipsel_24kc`。在 IDA 中按 MIPS Little-Endian 加载 `.ko`，找到模块入口 `init_module`。

### 还原 `init_module`

反编译结果虽然包含为 43 字节字符串做分块复制的噪声，但真正的数据流很短：

```c
const char *cipher =
    ">17;3-ee44`3`a{`boe{b2fb{4`d4{bdg5aoog4d44+";
char plain[44] = {0};

for (int i = 0; i < 43; ++i)
    plain[i] = cipher[i] ^ 0x56;

printk("%s", plain);
```

前半段的多次 `DWORD`、`WORD` 复制只是编译器把字符串常量搬到栈上的实现，不是另一层加密。后面的循环明确以索引遍历 43 字节，并逐字节异或常量 `0x56`。

用本地脚本复算：

```python
ciphertext = b">17;3-ee44`3`a{`boe{b2fb{4`d4{bdg5aoog4d44+"

assert len(ciphertext) == 43
plaintext = bytes(value ^ 0x56 for value in ciphertext)
print(plaintext.decode())
```

输出为：

```text
hgame{33bb6e67-6493-4d04-b62b-421c7991b2bb}
```

这也解释了模块为何调用 `printk`：若在原固件中正常加载，解密后的字符串会写入内核日志。静态恢复已经完整覆盖同一逻辑，没有运行未知模块的必要。

## 方法总结

- 固件题先确定容器、分区和文件系统，再在根文件系统中按服务名、包名、启动配置与可疑字符串收窄目标。
- OpenWrt 的 `/etc/modules-boot.d`、`/etc/modules.d` 和 `/usr/lib/opkg/info` 能把模块文件与启动行为、软件包名称对应起来。
- 反编译器展示的大块常量复制通常只是编译器优化或 ABI 产物；应沿最终使用点追踪数据，本题的有效算法只有单字节 XOR。
- 未知内核模块风险高。能静态复现的校验或解密逻辑不应通过实际加载模块来验证。
- 分类应依据决定性技术障碍：普通固件逻辑与内核模块行为恢复属于 Reverse，不能仅因载体是路由器固件就机械归入 Hardware。
