# Ports

## 题目简述

外层 `ports.zip` 包含 65335 个内层 ZIP，文件名为
`port-N.txt.zip`。每个内层 ZIP 的口令就是十进制端口号 $N$，解开后文本会指向下一个端口；最终端口的文本不再给数字，而是给出 flag。

这不是端口扫描题，也不需要尝试 65335 个未知口令。文件名已经提供每个档案的精确口令，核心是沿单链表式引用自动重建路径。

## 解题过程

从任意非终点端口开始均可沿其所在链走到终点。以 `port-1.txt.zip` 为例，口令为
`1`，其内容第一行是：

```text
Go to port 21105 instead :(
```

于是下一步读取 `port-21105.txt.zip` 并使用口令 `21105`。重复这一过程，直到第一行不再匹配数字端口。

由于最终档案使用 7-Zip 创建的 AES ZIP 方法，Python 标准库可能在终点报
`compression method 99 is not supported`。使用 7-Zip 命令行处理所有内层档案更稳定：

```python
import pathlib
import re
import subprocess
import zipfile

outer_path = pathlib.Path("ports.zip")
work = pathlib.Path("work")
work.mkdir(exist_ok=True)

with zipfile.ZipFile(outer_path) as outer:
    current = 1

    while True:
        member = f"ports/port-{current}.txt.zip"
        inner_zip = work / f"port-{current}.txt.zip"
        inner_zip.write_bytes(outer.read(member))

        subprocess.run(
            [
                "7z", "e", "-y",
                f"-p{current}",
                f"-o{work}",
                str(inner_zip),
            ],
            check=True,
            stdout=subprocess.DEVNULL,
        )

        text_path = work / f"port-{current}.txt"
        text = text_path.read_text(encoding="utf-8")

        match = re.search(r"Go to port (\\d+) instead", text)
        if not match:
            print(text)
            break

        current = int(match.group(1))
        text_path.unlink()
        inner_zip.unlink()
```

从端口 1 出发，链在 1759 个节点后到达终点端口 `42237`。使用口令
`42237` 解开该档案，得到：

```text
UMDCTF{dDSA-d_23+t0ta11y_n0t_NSFW_tCp_pAcKET-0_0-15039254&((*#@!}
```

随机填充消息只是增加文本长度，不参与下一端口解析。

## 方法总结

- 核心技巧：把数万个加密 ZIP 视为“端口号为节点、正文指向下一节点”的单链表，按已知口令逐节点遍历。
- 识别信号：档案名、口令和正文引用使用同一个整数命名空间时，应建立状态机，而不是对全部文件做无差别解压和全文搜索。
- 复用要点：解析下一端口时只匹配固定句式；终点可能使用 Python `zipfile` 不支持的 AES ZIP 方法，统一调用 7-Zip 可避免在最后一步失败。
