# Symbolic Needs 1

## 题目简述

附件是一份 Linux 内存镜像。目标是识别系统版本，为 Volatility 3 准备匹配的 Linux 符号表，再从内存中的 Bash 历史记录恢复一串点号分隔的十进制 ASCII。

题目仓库已经附带可用的 `ubuntu22.json`，但理解符号表为何必须与内核精确匹配，才能稳定复现后续插件输出。

## 解题过程

先用 `banner` 扫描内核标识：

```console
python3 vol.py -f dump.mem banner
```

结果中的关键信息为：

```text
Ubuntu 22.04
Linux 5.15.0-43-generic
5.15.0-43.46 amd64
```

Volatility 3 的 Linux 插件需要从对应内核调试信息生成 JSON 符号表。与该镜像匹配的包是：

```text
linux-image-unsigned-5.15.0-43-generic-dbgsym
版本 5.15.0-43.46 amd64
```

从调试包中取出：

```text
usr/lib/debug/boot/vmlinux-5.15.0-43-generic
```

再用 `dwarf2json` 生成：

```console
dwarf2json linux \
  --elf vmlinux-5.15.0-43-generic \
  > ubuntu22.json
```

把 `ubuntu22.json` 放入：

```text
volatility3/volatility3/symbols/linux/
```

仓库的 `solution/symbols.zip` 已包含同名 JSON，可以省去下载调试包和转换步骤。

符号加载成功后执行：

```console
python3 vol.py -f dump.mem linux.bash
```

插件恢复出 PID 1863 的一条命令：

```text
72.48.117.53.84.48.110.95.119.51.95.52.114.51.95.49.110.33.33.33
```

每段都是一个十进制 ASCII 码：

```python
encoded = (
    "72.48.117.53.84.48.110.95.119.51."
    "95.52.114.51.95.49.110.33.33.33"
)

decoded = "".join(
    chr(int(value))
    for value in encoded.split(".")
)
print(decoded)
```

输出：

```text
H0u5T0n_w3_4r3_1n!!!
```

按题目要求包上 flag 格式：

```text
SEKAI{H0u5T0n_w3_4r3_1n!!!}
```

## 方法总结

Linux 内存取证的首要工作是准确识别内核并准备匹配符号。只知道发行版名称不够，版本、ABI 和构建号都应一致。符号就绪后，优先检查 shell 历史、进程参数和打开文件等高价值证据；本题的编码层很浅，真正的门槛是让 Volatility 正确解释镜像结构。
