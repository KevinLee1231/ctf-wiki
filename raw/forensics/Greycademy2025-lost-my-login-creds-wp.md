# Greycademy2025 Lost my login creds

## 题目简述

附件是一台虚拟机的 OVA，题面称登录凭据保存在 `/root/flag.txt`，但又忘记了系统登录密码。关键是认识到 OVA 是虚拟机分发容器：只要客体文件系统没有加密，就不必先登录操作系统。

## 解题过程

将 OVA 导入 VirtualBox 或 VMware，或者把它当作 TAR 容器展开。展开后会得到虚拟机描述文件和 VMDK 虚拟磁盘：

```bash
tar -tf challenge.ova
mkdir unpacked
tar -xf challenge.ova -C unpacked
```

接着在 FTK Imager 中选择 `File -> Add Evidence Item -> Image File`，加入展开出的 `.vmdk`。也可以用支持 VMDK 的其他只读磁盘取证工具，重点是浏览客体文件系统，而不是启动虚拟机并尝试爆破口令。

在分区树中定位根文件系统，打开：

```text
/root/flag.txt
```

文件内容为：

```text
grey{wh0_n33d5_1091n_c23d3n71415}
```

仓库只保留了官方解法说明，4 GiB 左右的 OVA 仍通过外部链接分发，未随源码仓库归档；因此上述步骤来自官方解题链，当前本地复核范围止于容器与 VMDK 的取证方法，没有虚构额外分区编号或界面截图。官方材料还给出备用系统口令，但主线不需要使用它。

## 方法总结

“没有登录密码”不等于“无法读取磁盘”。操作系统认证保护的是正常登录路径，而未加密的离线文件系统可以由取证工具直接挂载或浏览。处理 OVA 时先识别容器层，再对 VMDK 做只读分析；这样既绕开无关的口令问题，也比启动客体系统更能保持证据状态。
