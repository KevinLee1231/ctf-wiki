# Virtual Love_Revenge(2.0)

## 题目简述

本题是 VMware 虚拟机修复 + 隐写 + Linux 单用户模式取证题。附件包含被加密且被破坏的 VMware 虚拟机和加密压缩包；需要先从日志中的零宽字符隐写提取字典，爆破 `.vmx` 加密密码，再修复 `.vmx` 和 `.vmdk` 结构，使 CentOS 虚拟机可启动。

`ReadMe.md` 未提供 Virtual Love 的可访问附件地址，本篇主要依据官方总 WP。后续进入虚拟机后，用 guest 用户发现假 flag 和 `Try to enter root!` 提示；通过 CentOS 单用户模式改 root 密码，进入 root 目录拿压缩包密码，再结合目标文件名 MD5 得到 2.0 最后一段 flag。

## 解题过程

题目描述给出 `Login User: guest`，附件解压后包含一个加密压缩包和一套 VMware 虚拟机文件。双击 `.vmx` 时提示虚拟机被加密，因此第一步是恢复 `.vmx` 加密密码。可以使用 [`pyvmx-cracker`](https://github.com/axcheron/pyvmx-cracker) 这类 VMware 配置爆破工具，但默认字典无法命中，需要先从附件中找字典。

先确认 VMware 目录中文件的职责，避免盲目替换。`.vmx` 是启动配置，`.vmdk` 是虚拟磁盘，`.nvram` 保存 BIOS 状态，`.vmsd` 保存快照元数据，`.vmxf` 是补充配置，`.log` 用于记录虚拟机活动。VMware 文件用途可参考 <https://blog.csdn.net/qq_38145502/article/details/103629709>。本题 `.vmxf` 中出现异常文本：

```text
You hurt my heart deeply! So I will revenge, I will destroy everything you have!
```

正常 `.vmxf` 不应包含这类句子，说明配置或磁盘文件被人为破坏。附件里还有三个 log 文件，最大 log 用十六进制编辑器或 Vim 查看时能看到不可见字符，只由 `\xe2\x80\x8d` 和 `\xe2\x80\x8c` 组成，也就是零宽连接符/非连接符。把两类字符映射成 0/1 后即可恢复字典，同时还能输出去掉隐写字符后的干净 log：

```python
import libnum

with open("vmware-1.log", "rb") as f, \
     open("dic.txt", "w") as dic, \
     open("log.txt", "wb") as clean:
    data = f.read()
    clean.write(data.replace(b"\xe2\x80\x8d", b"").replace(b"\xe2\x80\x8c", b""))

    for line in data.splitlines():
        line = line.replace(b"\xe2\x80", b"")
        bits = ""
        for ch in line:
            if ch == 0x8d:
                bits += "0"
            elif ch == 0x8c:
                bits += "1"
            else:
                break
        if bits:
            dic.write(libnum.b2s(bits).decode(errors="ignore") + "\n")
```

使用提取出的 `dic.txt` 爆破 `.vmx`，得到密码 `kx4s3a`。移除加密后打开虚拟机，会提示客户机操作系统未指定。检查 ISO 可确认系统是 CentOS，因此先把客户机系统改成 Linux CentOS 7 64 位。再次启动后出现 `Operating System not found`，并且 VMware GUI 中很多配置缺失，说明 `.vmx` 仍然损坏。

`.vmx` 配置项含义可对照 <http://sanbarrow.com/vmx.html>、VMware 相关手册 <https://cdn.ttgtmedia.com/searchVMware/downloads/RULE_CH09.pdf> 和修复记录 <https://blog.csdn.net/shuideyidi/article/details/40688369>。本题 `.vmx` 中出现大量 `XXX` 占位，例如：

```text
.encoding = "GBK"
displayName = "VirtualLove"
config.version = "8"
virtualHW.version = "16"
usb.vbluetooth.startConnected = "XXX"
tools.syncTime = "XXX"
vcpu.hotadd = "XXX"
```

`XXX` 不是合法配置值。修复方法是在干净 log 中找到以下两行之间的配置快照：

```text
2021-02-20T11:50:10.598+08:00| vmx| I125: DICT --- CONFIGURATION
...
2021-02-20T11:50:10.598+08:00| vmx| I125: DICT --- USER DEFAULTS
```

把中间配置复制回 `.vmx`，去掉前缀时间、日志字段和行尾隐写字符，并整理为正常左对齐配置：

```text
config.version = "8"
virtualHW.version = "16"
pciBridge0.present = "TRUE"
pciBridge4.present = "TRUE"
...
usb:0.present = "TRUE"
usb:0.deviceType = "hid"
usb:0.port = "0"
usb:0.parent = "-1"
```

修好 `.vmx` 后如果出现“指定的文件不是虚拟磁盘”，问题就转向 `.vmdk`。VMDK descriptor 和 extent 文件结构可参考 VMware VMDK 文档 <https://www.vmware.com/app/vmdk/?src=vmdk> 以及 libvmdk 的格式说明 <https://github.com/libyal/libvmdk/blob/main/documentation/VMWare%20Virtual%20Disk%20Format%20(VMDK).asciidoc>。本题共有 7 个 VMDK：一个较小 descriptor 和六个编号较大的 extent。

小 descriptor 文件由三部分组成：

```text
# Disk DescriptorFile
# Extent description
# The Disk Data Base
#DDB
```

对比正常 VMDK 后，可发现 descriptor 缺少 `version`、`parentCID` 和磁盘 extent 编号。`version` 默认为 `1`；本题没有快照，`parentCID` 可填 `ffffffff`；extent 按顺序补 `s001` 到 `s006`。

编号 1 到 6 的 extent 文件固定头部为 512 字节，关键字段包括 `KDMV` magic、version、flags、capacity、grain size、grain table/directory、dirty 标志和行结束符检测字节。flags 字段可对照 libvmdk 的说明：<https://github.com/libyal/libvmdk/blob/main/documentation/VMWare%20Virtual%20Disk%20Format%20(VMDK).asciidoc#vmdk_extent_file_flags>。本题这些 extent 缺少开头 8 字节的 signature 和 version，补：

```text
4B 44 4D 56 01 00 00 00
```

继续检查 offset 73 到 76 的传输损坏检测字节，也缺少 `\n`、空格、`\r`、`\n`，补：

```text
0A 20 0D 0A
```

保存后虚拟机即可正常启动。用题目给出的 `guest` 用户登录后，当前目录有假 flag，history 中有提示：

```text
Try to enter root!
```

`guest` 无权进入 `/root`，且 sudo 版本没有可直接利用的提权漏洞。CentOS 可通过单用户模式修改 root 密码：重启后在 GRUB 界面按 `e`，找到 `linux16` 开头的启动项，把 `ro` 改成：

```text
rw init=/sysroot/bin/sh
```

按 `Ctrl+x` 进入单用户模式后执行：

```bash
chroot /sysroot
LANG=en
passwd root
touch /.autorelabel
reboot -f
```

重启后使用新密码登录 root，在 root 目录下拿到压缩包密码：

```text
Revenge: f5`FU2)I$F0Oc'qL@pP)S
Revenge2.0: 2F!,O<DJ+@<*K0@<6L(Df-
```

目录中的 `7@rget-f1le` 取小写 MD5，是 2.0 flag 的最后一段。解压后得到：

```text
Revenge: d3ctf{Vmw@RE_1s_5oooo_C0mpl3x_ec4bb60e58}
Revenge2.0: d3ctf{Vmw@RE_1s_5ooo_C0mpl3x_8cf8463b34caa8ac871a52d5dd7ad1ef}
```

## 方法总结

- 核心技巧：先恢复虚拟机配置和磁盘格式，再用系统恢复模式进入目标系统，最后处理压缩包密码和隐写线索。
- 识别信号：VMware 目录中出现异常 `.vmxf` 文本、额外 log 文件带零宽字符、`.vmx` 中大量 `XXX` 配置值、`.vmdk` descriptor/header 缺字段。
- 复用要点：VMware 题要熟悉 `.vmx`、`.vmdk`、`.nvram`、`.vmsd`、`.log` 等文件职责；修复时可从日志中的 `DICT --- CONFIGURATION` 恢复配置，从 VMDK 文档核对 magic、version、换行校验字段。

