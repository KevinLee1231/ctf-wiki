# Lost my login creds

## 题目简述

题目提供一台虚拟机的 OVA 镜像，flag 位于来宾系统的 `/root/flag.txt`，但登录凭据未知。目标不是破解密码，而是离线提取虚拟磁盘并直接读取未加密文件系统。

## 解题过程

OVA 本质上是打包的虚拟机档案，直接对压缩容器运行 `strings` 不能可靠恢复文件。可以将 OVA 导入 VirtualBox/VMware，让软件展开配套文件；也可以把它作为 tar 归档解包：

```bash
tar -xf challenge.ova
```

展开后定位 `.vmdk` 虚拟磁盘，将其添加到 FTK Imager；Autopsy 或其他支持 VMDK 的磁盘取证工具也可完成同样操作。展开 Linux 文件系统，导航到：

```text
/root/flag.txt
```

文件系统没有加密，因此读取磁盘不需要来宾系统账户密码。文件内容为：

```text
grey{wh0_n33d5_1091n_c23d3n71415}
```

## 方法总结

虚拟机登录控制只保护正在运行的来宾系统，不等于磁盘静态加密。面对 OVA，应先拆出 VMDK，再按磁盘镜像处理；只有启用了全盘或文件级加密时，登录凭据才会成为离线读取的必要条件。
