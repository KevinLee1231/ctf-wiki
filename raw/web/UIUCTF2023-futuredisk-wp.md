# UIUCTF 2023 futuredisk Writeup

## 题目简述

服务端提供 `haystack.bin.gz`，标称大小接近 8 EiB，下载速度还被限制在约 4 KiB/s。解压后的内容几乎全为零，flag 被插在某个未知位置。完整下载无论从时间还是存储上都不可行。

突破口是 HTTP 响应支持 `Accept-Ranges: bytes`。后端 FUSE 文件系统能按偏移即时生成对应的 gzip/DEFLATE 数据，因此可以用 Range 请求随机读取少量压缩流，再根据零数据的周期性压缩结构二分定位 flag。

## 解题过程

### 确认随机访问能力

先查看响应头和文件末尾的小范围内容：

```bash
curl -I 'https://futuredisk-web.chal.uiuc.tf/haystack.bin.gz'
curl -r 9223372036854775700-9223372036854780899 \
  'https://futuredisk-web.chal.uiuc.tf/haystack.bin.gz' -o sample.bin
```

`206 Partial Content` 与 `Content-Range` 说明服务器接受任意字节区间，而且读取靠近 8 EiB 末尾的数据仍能很快返回。于是文件大小只是地址空间，不代表后端真的保存了 8 EiB。

最小 Range 读取函数如下：

```python
import requests

URL = "https://futuredisk-web.chal.uiuc.tf/haystack.bin.gz"

def read_range(start: int, size: int = 5120) -> bytes:
    end = start + size - 1
    r = requests.get(URL, headers={"Range": f"bytes={start}-{end}"})
    r.raise_for_status()
    assert len(r.content) == size
    return r.content
```

### 利用 DEFLATE 周期

DEFLATE 按位编码，并且每个字节内最低位先出现。分析文件开头可识别动态 Huffman 块；初始块负责产生第一个零，后续零块几乎全部由“从距离 1 复制 258 字节”的 match 组成。稳定块包含 32767 个 match，其压缩位长为：

$$
L=2\times32767+100=65634\text{ bits}.
$$

其中大量重复位表现为 `01`，落到字节上常见为 `0x55`。从一个已知位于 flag 之前的偏移向后扫描，连续确认多个块边界，即可测出 65634 位的周期；再在文件尾附近取得一个已知位于 flag 之后的样本。flag 自身使用一个不同的 DEFLATE 块，会破坏原周期，并使后续块相对原参考发生相位变化。

以稳定块长度为步长二分：在中点下载约 160 位，将实测位串与“该偏移本应出现的周期片段”比较。匹配与否分别对应扰动的一侧，持续缩小区间，约 $\log_2(8\text{ EiB}/L)$ 次请求就能逼近异常块，而不是线性扫描整个文件。

处理位流时不能直接使用普通大端二进制字符串。一个简单转换是：

```python
def deflate_bits(data: bytes) -> str:
    # DEFLATE：每个字节内 LSB first
    return "".join(f"{b:08b}"[::-1] for b in data)
```

### 对齐并解压 flag 块

二分到第一个异常周期附近后，再下载一个块大小左右的窗口。flag 块未必从字节边界开始，因此需要逐位移动起点，按 DEFLATE 的位序重新打包，并以 raw DEFLATE 模式尝试解压：

```python
import zlib

def pack_lsb(bits: str) -> bytes:
    usable = len(bits) // 8 * 8
    return bytes(int(bits[i:i + 8][::-1], 2)
                 for i in range(0, usable, 8))

bits = deflate_bits(read_range(candidate_offset, 9000))
for shift in range(len(bits)):
    try:
        plain = zlib.decompressobj(wbits=-zlib.MAX_WBITS).decompress(
            pack_lsb(bits[shift:])
        )
    except zlib.error:
        continue
    if b"uiuctf{" in plain:
        print(plain.rstrip(b"\0"))
        break
```

官方求解脚本通过上述周期二分得到异常块编号，并在附近删去尾部连续 `01` match 位后做同样的位偏移枚举，最终解出：

```text
uiuctf{binary search means searching a binary stream, right :D}
```

## 方法总结

本题的 Web 原语是 HTTP Range：它把不可下载的超大对象变成可随机访问的远程数组。随后需要理解 DEFLATE 的 LSB-first 位序、零流的稳定 match 块以及插入 flag 后的周期扰动。面对巨大稀疏或虚拟文件，应先验证是否支持常数时间 seek，再寻找可预测背景和异常判定函数；只要能回答“目标在中点之前还是之后”，文件的逻辑大小就不再是主要障碍。
