# oh-my-linux

## 题目简述

附件是 Linux 内存镜像。需要为其内核制作 Volatility 2 Profile，恢复用户文件系统，再从 Zsh 主题、加密压缩包和自定义别名三处取回 flag 片段。

## 解题过程

### 识别系统并制作 Profile

先从镜像字符串中定位内核构建信息：

```bash
strings oh-my-linux.mem | grep 'Linux version'
```

关键结果为：

```text
Linux version 5.10.0-21-amd64 ... #1 SMP Debian 5.10.162-1 (2023-01-21)
```

该内核来自 Debian 11。Volatility 2 的 Linux 插件依赖与目标内核完全匹配的类型信息和 `System.map`，因此应在 Debian 11 环境安装 `linux-image-5.10.0-21-amd64`、对应 headers、`build-essential` 与 `dwarfdump`，再从 [Volatility 2 官方仓库](https://github.com/volatilityfoundation/volatility) 构建 `module.dwarf`：

```bash
cd volatility/tools/linux
make
zip Linuxdebian51021x64.zip \
    module.dwarf \
    /boot/System.map-5.10.0-21-amd64
mv Linuxdebian51021x64.zip ../../volatility/plugins/overlays/linux/
```

如果本机软件源已不再提供该旧内核，需要从 Debian 归档取得同版本内核、headers 和 `System.map`；只使用“同为 5.10”的近似版本并不可靠。

### 恢复 Zsh 相关证据

Profile 可用后，`linux_bash` 能看到 PID 1014 的命令从 Bash 切换到 Zsh：

```bash
python2 vol.py -f oh-my-linux.mem \
    --profile=Linuxdebian51021x64 linux_bash
```

关键记录如下，PID 1014 最后执行 `zsh`，与后续恢复出的 Zsh 文件相互印证：

```text
Pid   Name  Command Time                   Command
1014  bash  2023-02-25 15:45:08 UTC+0000  echo $SHELL
1014  bash  2023-02-25 15:45:23 UTC+0000  echo "Bash doesn't feel as good as zsh"
1014  bash  2023-02-25 15:45:27 UTC+0000  echo "lol"
1014  bash  2023-02-25 15:45:29 UTC+0000  zsh
```

随后恢复镜像中仍可重建的文件：

```bash
python2 vol.py -f oh-my-linux.mem \
    --profile=Linuxdebian51021x64 \
    linux_recover_filesystem --dump-dir=files/
```

`/home/mylinux/.zsh_history` 给出了完整取证路线。去掉时间戳后，关键命令为：

```bash
echo $SHELL
sh -c "$(curl -fsSL https://gitee.com/mirrors/oh-my-zsh/raw/master/tools/install.sh)"
mv mysecretflag1.zsh-theme ~/.oh-my-zsh/themes
vim ~/.zshrc
source ~/.zshrc
zip -o flag2.zip flag2.txt -P $SHELL
rm -rf flag2.txt
G1v3m3F14ggg3
sudo ./avml oh-my-linux.mem
```

这组记录表明：用户通过 Gitee 上的 Oh My Zsh 镜像安装脚本配置 Zsh，把 `mysecretflag1.zsh-theme` 移入主题目录；随后用 `$SHELL` 作为密码创建 `flag2.zip`，并执行自定义命令 `G1v3m3F14ggg3`。

第一段位于 `.oh-my-zsh/themes/mysecretflag1.zsh-theme` 的注释中：

```text
flag1: flag{a4fca17e-
```

历史记录中的压缩命令是 `zip -o flag2.zip flag2.txt -P $SHELL`。当时 `$SHELL` 为 `/usr/bin/zsh`，据此解压桌面的压缩包：

```bash
unzip -P /usr/bin/zsh flag2.zip
```

第二段为：

```text
flag2: 3d30-47f3-b79c
```

最后检查恢复出的 `.zshrc`，可以看到 `G1v3m3F14ggg3` 是一个别名，其命令把内嵌 Base64 数据交给 `base64 -d`。提取并解码该数据后，ASCII Art 表示：

```text
flag3: -147e9c043850}
```

按顺序拼接三段，最终得到：

```text
flag{a4fca17e-3d30-47f3-b79c-147e9c043850}
```

## 方法总结

Linux 内存取证首先要精确匹配内核 Profile，随后再围绕进程历史建立证据链。本题中 `.zsh_history` 不是 flag 本身，却把三个载体、压缩密码来源和别名入口串联起来；恢复文件后应按命令发生顺序解释每条线索，最后再拼接片段。
