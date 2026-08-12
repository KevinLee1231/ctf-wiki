# DownUnderCTF 2021 - Rabbit

## 题目简述

附件名为 `flag.txt`，但题面提示内容被埋了约 1000 层。生成脚本在每一层随机选用 ZIP、gzip、bzip2 或 XZ 压缩，然后又把产物改回原文件名。因此，扩展名始终不可信，必须依据每层文件的实际魔数反复识别并解包。

## 解题过程

先用 `file` 查看最外层真实格式，而不是直接把附件当文本打开：

```bash
file flag.txt
```

然后循环处理当前唯一文件。gzip、bzip2 和 XZ 解压工具通常依赖扩展名，可以临时加上正确后缀再解压；ZIP 解包后会同时保留压缩包，所以要删掉已经消费的外层文件。一个紧凑的处理框架如下：

```bash
while true; do
    kind=$(file -b flag.txt)
    case "$kind" in
        *"gzip compressed"*)
            mv flag.txt flag.txt.gz
            gunzip flag.txt.gz
            ;;
        *"bzip2 compressed"*)
            mv flag.txt flag.txt.bz2
            bunzip2 flag.txt.bz2
            ;;
        *"XZ compressed"*)
            mv flag.txt flag.txt.xz
            unxz flag.txt.xz
            ;;
        *"Zip archive data"*)
            mv flag.txt outer.zip
            unzip -q outer.zip
            rm outer.zip
            ;;
        *"ASCII text"*)
            break
            ;;
        *)
            printf 'unknown layer: %s\n' "$kind" >&2
            exit 1
            ;;
    esac
done
```

为了避免归档内文件名变化造成脚本失效，实际自动化时可以在每轮确认目录中只有一个文件，再把它规范为 `flag.txt` 后继续。重复约 1000 次后，最内层成为 ASCII 文本；它仍是 Base64 表示，最后执行：

```bash
base64 -d flag.txt
```

得到：

```text
DUCTF{babushkas_v0dka_was_h3r3}
```

## 方法总结

本题考查基于文件特征而非扩展名的递归取证。ZIP、gzip、bzip2、XZ 都有稳定魔数，`file` 可以在每层重新识别。可靠脚本还应检查每轮输入数量、未知类型和解压失败状态，避免在某层产生多个文件后误删证据；最终出现文本也不代表结束，还要根据内容判断是否存在 Base64 等表示层编码。
