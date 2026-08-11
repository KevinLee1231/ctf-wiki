# mkductfiso

## 题目简述

附件是一个无法启动的 Arch Linux ISO。出题源码显示其被重新打包时故意删除了 `initramfs-linux.img`，并植入替换过的 `/usr/bin/grep`；官方解答则给出修复启动介质的路径。任务的主要工作是从 ISO 文件系统恢复启动证据并修复归档，归入 Forensics。

## 解题过程

先解开 ISO，检查启动项配置。官方解答指出 `archiso_sys-linux.cfg` 中额外的 `initrd` 调用会阻止启动；可以删除这项错误调用。另一种等价修复是从一份同日期、同架构的正常 Arch ISO 取出 `arch/boot/x86_64/initramfs-linux.img`，放回解包目录中的相同位置。

随后以原始的 El Torito 引导信息重新制作 ISO。关键不是复制一个普通压缩包，而是保留 `isolinux.bin`、`boot.cat`、isohybrid MBR 与根目录树，否则镜像仍不能正常引导。修复后的系统会进入带有篡改版 `grep` 的环境。

源码给出了触发条件：生成器以 `accessibility=/proc/cmdline` 作为第一段已知输入，构造加密后的 flag；构建脚本把 `grep-backdoored` 覆盖进 squashfs。因此在系统中以固定字符串方式查询内核命令行即可触发该二进制的后门逻辑：

```bash
grep -Fqa 'accessibility=' /proc/cmdline
```

无需猜测，题目配置和生成器中的 flag 常量交叉确认答案为：

```
DUCTF{D4Mn_YuO_R34LLy_G0t-tH4t-T0_Bo0t?!#<3}
```

## 方法总结

损坏启动镜像的排查应分层进行：先确认启动配置是否引用了不存在或重复的 initramfs，再核验 ISO 的引导目录与 El Torito 元数据，最后才进入 rootfs 观察可执行文件差异。这里自定义 `grep` 是修复后才会暴露的证据；若只盯着后门而不恢复镜像启动路径，就无法完成题目要求的完整链条。
