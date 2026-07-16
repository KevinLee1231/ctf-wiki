# NTFS很ez啦

## 题目简述

附件 `0xGame3.vhd` 是一块使用 GPT 分区表的虚拟磁盘。镜像中的 Basic data partition 从扇区 `65664` 开始，文件系统为 NTFS，卷标为 `0xGame`。直接查看根目录只能看到 `I5hXl5Qq9rljnACj.dat` 等随机文件名，真正的 flag 片段已经随重命名操作从当前目录项中消失。

本题的关键是 NTFS 的事务日志 `$LogFile`。NTFS 会把影响卷结构的事务写入该文件，以便系统故障时回滚；因此，即使文件已经改名，相关的旧文件名仍可能留在日志记录中。题目把 flag 分片编码成十进制整数文件名，再将这些文件重命名为随机 `.dat` 文件，需要按日志顺序恢复旧名称并解码。

## 解题过程

### 1. 定位 NTFS 分区与 `$LogFile`

先用 Sleuth Kit 查看分区表和 NTFS 根目录：

```bash
mmls 0xGame3.vhd
fls -o 65664 0xGame3.vhd
```

`mmls` 给出的目标分区范围为 `65664`～`405631` 扇区；`fls` 则确认根目录中存在 MFT 记录号为 `2` 的 `$LogFile`。可以在 Autopsy 中导出该文件，也可以直接使用 `icat`：

```bash
icat -o 65664 0xGame3.vhd 2 > LogFile
```

实测该 `$LogFile` 的大小为 `2097152` 字节。根目录中能看到五个随机命名的 `.dat` 文件，但看不到后续用于解码的十进制文件名，说明必须转向历史事务记录。

### 2. 解析重命名记录

[LogFileParser](https://github.com/jschicht/LogFileParser) 可以解码 NTFS `$LogFile` 中的事务和属性变化，并把主结果写入 CSV，同时导入 `ntfs.db`。它还会把嵌在 `$LogFile` 中的 USN Journal 记录解码到相应字段，因此可以直接查询 `lf_UsnJrlReason`、`lf_FileName` 和 `lf_LSN`，无需手工解释二进制日志结构。

使用 LogFileParser 处理导出的日志后，在 SQLite 中筛选重命名事件，并按 LSN 排序：

```sql
SELECT
    lf_LSN,
    lf_UsnJrlTimestamp,
    lf_UsnJrlReason,
    lf_FileName
FROM LogFile
WHERE lf_UsnJrlReason LIKE '%RENAME%'
ORDER BY lf_LSN;
```

同一次重命名通常会出现 `RENAME_OLD_NAME` 与 `RENAME_NEW_NAME`。题目要求关注前后文件名变化，因此只取旧名称中形如十进制整数的部分，并保持 LSN 顺序，得到：

```text
13643046854681979.txt
6426765720352224837.txt
6876554908428166514.txt
28539333146592819.txt
555819389.txt
```

这里不能按整数大小或文件名的字典序重新排序；LSN 才表示这些事务在日志中的先后关系，也就是 flag 分片的拼接顺序。

### 3. 将十进制整数还原为字节串

去掉 `.txt` 后，将每个十进制整数按大端序转换为最短字节串，再以 UTF-8 解码。所需字节数可写为 `$L=(n.bit\_length()+7)//8$`：

```python
numbers = [
    13643046854681979,
    6426765720352224837,
    6876554908428166514,
    28539333146592819,
    555819389,
]


def integer_to_text(number: int) -> str:
    length = max(1, (number.bit_length() + 7) // 8)
    return number.to_bytes(length, byteorder="big").decode("utf-8")


flag = "".join(integer_to_text(number) for number in numbers)
print(flag)
```

运行结果为：

```text
0xGame{Y0u_H@vE_nnAsTered_NTF3!!!}
```

因此，最终 flag 为：

```text
0xGame{Y0u_H@vE_nnAsTered_NTF3!!!}
```

## 方法总结

这道题利用了“当前文件系统状态”和“历史事务状态”的差异：随机 `.dat` 名称只是现状，真正的信息仍保留在 `$LogFile` 的重命名记录里。解题时应先确定正确的分区偏移，再解析 `$LogFile`，按 LSN 保持事件顺序，并区分 `RENAME_OLD_NAME` 与 `RENAME_NEW_NAME`。恢复出的长十进制文件名不是普通编号，而是大端字节串的整数表示；按最短长度转回字节并顺序拼接即可得到 flag。
