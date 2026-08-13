# Greycademy2025 shy_zipper

## 题目简述

附件是一份 ZIP。正常解包看不到 flag，但题目提示 ZIP 会记录自身结构，同时结构之外仍可能存在数据。目标是解析 ZIP 的中央目录结束记录，并恢复追加在归档末尾的隐藏内容。

## 解题过程

ZIP 的 End of Central Directory（EOCD）签名是 `PK\x05\x06`。EOCD 固定部分长 22 字节，其中偏移 20 的两字节是可变注释长度。由此可以精确算出 ZIP 结构的结束位置，再检查其后的数据：

```python
import base64
import re

data = open("shy_zipper.zip", "rb").read()
sig = b"PK\x05\x06"
off = data.rfind(sig)
if off < 0:
    raise ValueError("EOCD not found")

comment_len = int.from_bytes(data[off + 20:off + 22], "little")
eocd_end = off + 22 + comment_len
trailing = data[eocd_end:]

for candidate in re.findall(rb"[A-Za-z0-9+/]+={0,2}", trailing):
    try:
        decoded = base64.b64decode(candidate, validate=True)
    except ValueError:
        continue
    if b"grey{" in decoded:
        print(decoded.decode())
```

生成端源码也印证了这一布局：它先创建合法 ZIP，然后在 EOCD 后追加空字节、`base64(flag)` 和更多填充。脚本输出：

```text
grey{s0m3_Th1Ng5_4r3_b3tT3r_L3fT_uNz1Pp3d}
```

也可以用 `strings` 偶然看到末尾 Base64，但那依赖载荷恰好是可打印文本，不能解释数据为什么位于那里；解析 EOCD 才是稳定主线。

## 方法总结

ZIP 解包器只消费受中央目录描述的成员，并不保证物理文件在 EOCD 后没有尾随数据。分析此类题时，应按格式字段计算逻辑结束位置，而不是从文件末尾硬编码偏移。`rfind` EOCD、解析小端注释长度、再对尾随区做有约束的 Base64 检测，既能复现，也能避免把归档内部的普通字符串误判为 flag。
