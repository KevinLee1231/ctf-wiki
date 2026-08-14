# bi0sCTF 2024 - ReAL-File-System

## 题目简述

题目给出一份被勒索程序处理、随后又被人为破坏的 ReFS 镜像。需要修复 SuperBlock/Checkpoint 的自引用 LCN，解析 ReFS redo-only 日志，恢复目录和文件操作时间线，雕刻删除文件，并逆向 `time_update.exe` 的时间派生密钥以解密八个 `.tort` 文件。服务依次校验六组列表，全部正确后返回 flag。

## 解题过程

### 修复 ReFS 元数据

VBR 中 `BytesPerSector=0x200`、`SectorsPerCluster=0x08`，所以：

$$
\text{cluster size}=0x200\times0x08=0x1000.
$$

ReFS 1.2 有三个 SuperBlock 和两个 Checkpoint；两类元数据页的 `LCN(1)` 都是指向自身的逻辑簇号。本镜像把这些字段清零，导致普通工具无法挂载。根据实际元数据签名位置恢复出的候选 LCN 为：

```text
SuperBlock: 0x00001e, 0x0f7ffd, 0x0f7ffe
Checkpoint: 0x0027d0, 0x01dc48
```

对目标页使用：

$$
LCN=\frac{\text{byte offset}}{0x1000}
$$

并以小端形式回填 `LCN(1)`。例如 LCN `0x1e` 对应字节偏移 `0x1e000`。公开题解正文曾把该偏移写成 `0x1e0000`，但紧接着的除法和 LCN 都对应 `0x1e000`；实际修复必须以镜像中的签名位置和结构解析结果为准。修复一组相互一致的 SuperBlock 与 Checkpoint 后，Active Disk Editor 即可识别卷。

### 解析 `MLog` 与目录操作

在 ReFS 的 LogFile 区域搜索 `MLog`，跳过前两个 control log page。ReFS 日志只记录 redo；redo record header 长 `0x38` 字节，后面是若干 `(offset, size)` 对和尾记录。按这些偏移切分同一事务，再以连续 opcode、目录/文件前缀和下一 `MLog` 页中的 FILETIME 关联操作。

目录重命名序列为：

```text
0x02 -> 0x02 -> 0x01 -> 0x01 -> 0x04
```

得到 Q1：

```python
[
    ["e88e52cac", "88077a4a1370", "2024-02-18 13:10:20.49"],
    ["fb828d071", "cad090f9724d", "2024-02-18 13:12:12.48"],
]
```

永久删除目录的序列为 `0x02 -> 0x0f -> 0x02 -> 0x0f -> 0x04 -> 0x12`；普通删除还可结合回收站 `$Ixxxxx` 元数据确认。Q2 为：

```python
[
    ["f8f1c218f9", "2024-02-18 13:11:28.57"],
    ["de34f60ab", "2024-02-18 13:13:19.60"],
    ["c062fb828", "2024-02-18 13:10:48.62"],
]
```

目录创建事务的关键序列是 `0x00 -> 0x00 -> 0x04 -> 0x10 -> 0x01 -> 0x01 -> 0x01 -> 0x0e -> 0x03 -> 0x04`。包括后来被重命名或删除的原目录，Q3 为：

```python
[
    ["de34f60ab", "2024-02-18 13:06:59.22"],
    ["e88e52cac", "2024-02-18 13:07:31.13"],
    ["fb828d071", "2024-02-18 13:08:13.88"],
    ["f8f1c218f9", "2024-02-18 13:08:40.58"],
    ["c062fb828", "2024-02-18 13:09:14.55"],
    ["bb6de6190", "2024-02-18 13:09:34.00"],
]
```

### 恢复删除文件

普通删除文件可以由 `$Ixxxxx` 取得原路径和删除时间，再按 `$Rxxxxx` 或数据 extent 雕刻内容；永久删除则依赖 LogFile 中的 extent。恢复出的三份删除文件及 MD5 为：

```text
f91488b7e00c31793bd7aa98c51896d0  simple-pass.txt
4c009b045056af8f9bb401c69408d2cf  19ff211f
c50c5bcb9e98537e3d63df1bc68a81d0  fe0c329
```

所以 Q4 提交上述三个哈希的单元素列表。文件删除时间线 Q5 为：

```python
[
    ["simple-pass.txt", "2024-02-18 13:15:00.51", "Simple"],
    ["19ff211f", "2024-02-18 13:14:31.43", "Simple"],
    ["fe0c329", "2024-02-18 13:15:52.49", "Simple"],
    ["ead47cb", "2024-02-18 13:19:26.69", "Permanent"],
    ["essay.txt", "2024-02-18 13:18:22.47", "Permanent"],
]
```

### 还原 `.tort` 文件名、时间与 extent

文件重命名事务的关键序列为：

```text
0x02 -> 0x05 -> 0x01 -> 0x04 -> 0x04
```

同一文件经历“原名 -> 随机名 -> 第二随机名 -> `.tort`”三次重命名；第二次重命名的伪造系统时间参与密钥生成。日志恢复结果如下：

```text
15005-39026.pdf            -> bf2f63b3 -> 0cf51fbc -> 0cf51fbc.tort  2010 2 2 26 12 38 43 0
binary-01.gif              -> c7982ef6 -> a917438f -> a917438f.tort  1995 2 2 27 02 11 42 0
Everest Vista.webp         -> d406327c -> 3a7fab71 -> 3a7fab71.tort  1990 8 2 28 21 06 35 0
Paranormal Phenomenon.docx -> 830c92a3 -> bb292337 -> bb292337.tort  1993 7 2 16 17 10 46 0
so-cappy.jpg               -> 141e0f79 -> 24819686 -> 24819686.tort  2001 4 2 19 08 27 45 0
stuffs.rar                 -> f15ebcd2 -> 7a6c7166 -> 7a6c7166.tort  2009 11 2 22 17 05 55 0
ySq12b0T.mp4               -> 86c66c9c -> 185c65f8 -> 185c65f8.tort  2007 7 2 28 03 27 33 0
vl36hkjkzbh91.png          -> 313feb6e -> cc876a3b -> cc876a3b.tort  1985 3 2 04 16 23 53 0
```

按元数据 extent 从镜像雕刻时要拼接非连续段。每个 `.tort` 的前 4 字节为标识，接下来 4 字节是小端有效载荷长度；应按该长度截掉簇对齐产生的尾部填充，实际密文从偏移 8 开始。

### 从时间派生 key 并枚举 nonce

`time_update.exe` 先把第二次重命名时刻作为 `SYSTEMTIME` 转换为 `FILETIME`，对低 32 位左旋 4 位、高 32 位左旋 3 位，再经逆向出的 16 轮变换与 30 字符映射生成 key 主体。可概括为：

```text
lo = rol32(FILETIME.low, 4)
hi = rol32(FILETIME.high, 3)
key_body = b30(hi) || b30(lo)
         || b30(F(lo + hi))
         || b30(F(lo * hi))
         || b30(F(lo & hi))
key1 = key_body || nonce4
key2 = md5(key1).hexdigest().encode()
```

其中 `b30` 使用字符表 `0123456789abcdef!#&*%GHIJ-lm+_`，并按样本循环从最低位余数开始依次追加字符，而不是输出常规高位在前的进制字符串；`F` 是样本中的 16 轮 TEA 风格 32 位变换。未知的 `nonce4` 也只从这 30 个字符中选取，因此每个文件最多枚举 $30^4=810000$ 个候选。

加解密均为同一个重复异或：

$$
P_i=C_i\oplus key1_{i\bmod |key1|}\oplus key2_{i\bmod |key2|}.
$$

用已恢复的原扩展名验证 PNG/JPEG/GIF/WEBP/RAR/ZIP/PDF 文件头；对短文件头产生的多解，再检查 JPEG EOF、ZIP/MP4 chunk 等结构。解密并去除额外字节后的 Q6 为：

```text
da8ed3e98eb5a2ba769ea60b48b0f6eb  15005-39026.pdf
d58621ce6e560ba1c045892aef0b5f8b  binary-01.gif
683092bd6640e62a3dc49b412da4fe71  Everest Vista.webp
11d9788ce48371a6ef230892ada1554d  Paranormal Phenomenon.docx
bc9a53c83976e9779bce2d0635f1bbbe  so-cappy.jpg
111fb8624db9365af79e6ec446b00eac  stuffs.rar
76675928a19bcc5602ef81c7a833d3fa  vl36hkjkzbh91.png
4d9c5a006c4315625c86d94a8fd9fd2e  ySq12b0T.mp4
```

全部六组答案通过后得到：

```text
bi0sctf{ReAL_1_w0nd3r_wHa7_t1m3_is_17_14dbc653fdb414c1d}
```

ReFS 结构截图、完整 opcode 样例和每个 extent 可在[官方题解](https://blog.bi0s.in/2024/03/12/Forensics/ReAL-File-System-bi0sCTF2024/)中核对；正文保留了复现所需的结构、操作序列、所有答案和密钥派生关系。

## 方法总结

本题分成三层：先用簇大小和元数据实际位置修复 SuperBlock/Checkpoint 自引用；再解析 redo-only `MLog`，以连续 opcode 还原目录与文件的创建、重命名和删除；最后把第二次重命名时间转换为 FILETIME，重建 key 主体并枚举四字节 nonce。雕刻结果必须按 `.tort` 头中的长度去除块填充，候选密钥也必须同时通过文件头、尾标记或容器 chunk 验证，不能只凭几字节 magic 判定。
