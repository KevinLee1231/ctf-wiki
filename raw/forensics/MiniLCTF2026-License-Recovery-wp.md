# License Recovery

## 题目简述

题目给出一个 OCI / Docker 镜像归档。最终文件系统中只有循环读取许可证的 `/app/check`，但镜像内的 `/README.txt` 和构建历史提示：真正用于恢复 flag 的 `/opt/tools/recover_flag.py` 曾出现在较早的 layer 中，随后被删除。Docker/OCI 分层删除通常只在新层写入 whiteout 标记，并不会从旧层抹去文件内容，因此第一步是沿镜像历史恢复该脚本。

恢复脚本仍需要一个 32 位 token。检查程序把候选 token 渲染成四只手组成的 PNG，再与二进制内嵌的目标 PNG 比较；每只手编码一个字节。完整解法因此包含两段证据链：从旧 layer 恢复解密脚本，再从 ELF 内嵌手势图读出 token。

## 解题过程

### 恢复被删除的 layer 文件

归档根目录存在 `manifest.json`、`index.json`、`oci-layout` 和 `blobs/sha256/`，可以据此确认它是镜像归档。`manifest.json` 按从旧到新的顺序列出各层；构建记录的关键步骤为：

```Dockerfile
COPY build/out/check /app/check
COPY build/run_check.sh /app/run_check.sh
RUN mkdir -p /opt/tools
COPY build/recover_flag.py /opt/tools/recover_flag.py
RUN printf '...Layer history matters more than the final filesystem...' > /README.txt
RUN rm /opt/tools/recover_flag.py
```

最后的 `rm` 只让脚本在合并视图中不可见。按 `manifest.json` 的层顺序检查各 blob，能在哈希以 `b130da0a` 开头的层中恢复它：

```bash
tar -xvf blobs/sha256/b130da0afd74ff186b806909ebfc7016aee1b1f8ebdfbe05be9c6a7abc1371c2

# 提取结果
opt/tools/recover_flag.py
```

同理，`check` 和 `run_check.sh` 分别位于哈希以 `83de1e27`、`03184cb8` 开头的历史层。也可以用 `dive` 查看每层文件变化，但工具界面不是解法依赖；关键是理解合并后的容器文件系统不会销毁旧层数据。

恢复出的脚本接受 `LICENSE-XXXXXXXX` 或八位十六进制数，将 token 解析成 32 位整数。它按以下方式生成密钥流：

```python
def derive_keystream(key: int, salt: bytes, length: int) -> bytes:
    key_bytes = key.to_bytes(4, "big")
    stream = bytearray()
    counter = 0
    while len(stream) < length:
        stream.extend(
            hashlib.sha256(
                key_bytes + salt + counter.to_bytes(4, "big")
            ).digest()
        )
        counter += 1
    return bytes(stream[:length])

plaintext = bytes(
    byte ^ keystream[index]
    for index, byte in enumerate(ciphertext)
)
```

解密后还会核对明文的 SHA-256，并要求结果以 `miniL{` 开头、以 `}` 结尾。因此脚本解决了“如何由 token 恢复 flag”，接下来只需确定 token。

### 从 `check` 的内嵌 PNG 读取 token

`strings` 能在 `check` 中找到提示和 [`llvm-jutsu`](https://github.com/thebabush/llvm-jutsu) 的来源地址；`binwalk` 则在文件偏移 `90368`（`0x16100`）识别出一张 $256\times256$ 的 RGBA PNG：

```bash
binwalk app/check
dd if=app/check of=hand-token.png bs=1 skip=90368
```

![四象限彩色手势图：四只手分别编码 32 位许可证 token 的四个字节](MiniLCTF2026-License-Recovery-wp/license-hand-token.png)

检查程序的渲染源码说明了准确位序：循环变量 $i=0,1,2,3$ 依次取

$$
\text{byte}_i=(\text{value}\gg 8i)\mathbin{\&}0xff,
$$

并按左上、右上、左下、右下放置，也就是最低有效字节在左上。每只手从拇指到其余七根指段依次表示 bit 7 到 bit 0；伸直为 1，弯曲为 0。读图得到：

| 位置 | 字节序号 | 二进制 | 十六进制 |
|---|---:|---:|---:|
| 左上 | byte 0 | `11110101` | `F5` |
| 右上 | byte 1 | `11101010` | `EA` |
| 左下 | byte 2 | `01001111` | `4F` |
| 右下 | byte 3 | `01101111` | `6F` |

按通常的高位在前形式重组为 `0x6F4FEAF5`，所以许可证为：

```text
LICENSE-6F4FEAF5
```

将它交给检查程序和旧层恢复出的脚本，两者均通过验证：

```text
$ ./app/check LICENSE-6F4FEAF5
License accepted.
The old recovery helper used the same 32-bit token.

$ python3 opt/tools/recover_flag.py LICENSE-6F4FEAF5
miniL{535a28d2-bcfa-4497-8519-f80443a056b4}
```

## 方法总结

- 核心技巧：根据 OCI 镜像 layer 顺序恢复被 whiteout 隐藏的旧文件，再解析 ELF 中内嵌的手势 PNG，按字节序还原 32 位 token。
- 识别信号：镜像最终文件系统缺少关键文件，但 `history`、README 或 layer 变化明确显示文件曾被复制后删除；这时应检查旧 blob，而不是只进入容器查看合并视图。
- 复用要点：恢复文件时记录 layer 顺序和原始路径；读自定义可视化整数时必须从生成/校验源码确认“图块顺序、位顺序、0/1 表示”，不能只凭图形猜端序。这里左上是最低有效字节，最终十六进制输出要反向按 byte 3 到 byte 0 书写。
