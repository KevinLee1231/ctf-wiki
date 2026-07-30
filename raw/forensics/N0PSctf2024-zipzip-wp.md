# ZIPZIP

## 题目简述

`archive.zip` 能够正常解压，表面文件是一份 ZIP 格式说明文档。按 ZIP 结构检查原始字节后，可以发现首个文件的压缩数据末尾与中央目录 `PK\x01\x02` 之间还夹着一段未被目录索引的数据。

这段数据具备本地文件头的全部字段，唯独缺少开头四字节签名 `PK\x03\x04`。真正的 flag 被作为第二个 ZIP 条目藏在这里；任务是补回本地头签名，并为它重建中央目录和目录结束记录。

## 解题过程

### 定位游离数据

ZIP 本地文件头包含压缩大小、文件名长度和额外字段长度。首条记录的实际结束位置为：

$$
30+\text{filename length}+\text{extra length}+\text{compressed size}
$$

从该位置继续读取，直到原归档的中央目录签名 `PK\x01\x02`，所得区域正好符合“缺少本地头签名的第二条记录”。

### 重建单文件 ZIP

下面的脚本逐字段解析附件，不依赖写死的文件名长度、CRC 或压缩大小。它只给隐藏记录补上签名，并使用同一组元数据创建中央目录：

```python
from pathlib import Path
import struct


LOCAL = struct.Struct("<4s5H3I2H")


def parse_local(data: bytes):
    fields = LOCAL.unpack_from(data)
    (
        signature,
        version,
        flags,
        method,
        mtime,
        mdate,
        crc32,
        compressed_size,
        uncompressed_size,
        name_length,
        extra_length,
    ) = fields
    assert signature == b"PK\x03\x04"

    name_start = LOCAL.size
    name = data[name_start:name_start + name_length]
    entry_end = (
        LOCAL.size
        + name_length
        + extra_length
        + compressed_size
    )
    return fields, name, entry_end


archive = Path("archive.zip").read_bytes()

# 跳过归档中可见的第一条记录。
_, _, first_entry_end = parse_local(archive)
central_offset = archive.find(b"PK\x01\x02", first_entry_end)
assert central_offset != -1

# 游离区域缺少四字节本地文件头签名。
hidden = b"PK\x03\x04" + archive[first_entry_end:central_offset]
fields, filename, hidden_end = parse_local(hidden)
(
    _signature,
    version,
    flags,
    method,
    mtime,
    mdate,
    crc32,
    compressed_size,
    uncompressed_size,
    name_length,
    _extra_length,
) = fields
local_entry = hidden[:hidden_end]

# 为隐藏条目构造中央目录。
central = struct.pack(
    "<4s6H3I5H2I",
    b"PK\x01\x02",
    0x0314,             # version made by
    version,
    flags,
    method,
    mtime,
    mdate,
    crc32,
    compressed_size,
    uncompressed_size,
    name_length,
    0,                  # central extra length
    0,                  # comment length
    0,                  # disk number
    0,                  # internal attributes
    0,                  # external attributes
    0,                  # local header offset
) + filename

# 单磁盘、单条目的 End of Central Directory。
eocd = struct.pack(
    "<4s4H2IH",
    b"PK\x05\x06",
    0,
    0,
    1,
    1,
    len(central),
    len(local_entry),
    0,
)

Path("recovered.zip").write_bytes(local_entry + central + eocd)
```

解压 `recovered.zip`，其中唯一文件的内容为：

```text
N0PS{z1p_z1p_z1p_z1p}
```

## 方法总结

- 核心技巧：依据首个 ZIP 条目的声明长度识别未被中央目录引用的游离记录，再补齐本地头和索引结构。
- 识别信号：归档能够正常解压，但文件数据末尾到中央目录之间存在额外字节；额外区域在偏移四字节后符合本地文件头字段布局。
- 复用要点：不要仅搜索 `PK` 签名，因为本题恰好删除了它。应以结构化长度计算边界，并用 CRC、压缩大小、文件名长度及最终可解压性共同验证重建结果。
