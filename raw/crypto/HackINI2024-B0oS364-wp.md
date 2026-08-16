# HackINI2024 B0oS364

## 题目简述

题目先对 flag 做 Base64 编码，再在每个编码字符后随机插入 0 至 3 个不属于标准 Base64 字母表的 Unicode 字符。目标是从混入大量噪声的文件中筛出真正的 Base64 数据并还原 flag。

## 解题过程

Base64 的有效数据字符只有以下 64 个：

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

题目生成器的噪声范围还可能产生 `=`，所以不能简单地保留输入中的所有等号；随机插入到数据中部的 `=` 会让解码提前结束或报错。正确做法是只保留 64 个数据字符，最后再根据长度统一补齐填充：

```python
import base64

alphabet = set(
    b"ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    b"abcdefghijklmnopqrstuvwxyz"
    b"0123456789+/"
)

with open("output.txt", "rb") as f:
    raw = f.read()

clean = bytes(byte for byte in raw if byte in alphabet)
clean += b"=" * (-len(clean) % 4)
print(base64.b64decode(clean).decode())
```

对附件执行后得到：

```text
shellmates{YooU_H4v3_7O_KNOW_B4s364_Ch4r4c73R2}
```

仓库中的官方 `solve.py` 将读取到的整数型字节与字符串列表比较，导致过滤条件失效；上面的实现修正了这一类型错误，同时避免误保留作为噪声出现的 `=`。

## 方法总结

这类题的关键不是“删除看起来奇怪的字符”，而是建立严格的白名单。二进制读取后应使用整数或字节集合做类型一致的成员判断，并把 Base64 的 `=` 当作末尾结构性填充重新生成。这样既能处理任意 Unicode 噪声，也不会被数据中部的伪填充干扰。
