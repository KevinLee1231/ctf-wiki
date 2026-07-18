# week3threeThousand

## 题目简述

附件 `3000.zip` 由 3000 层 ZIP 逐层嵌套而成，每一层都只有一个文件，解压密码是 `00`～`99` 中的一个两位数字。目标是自动枚举每层密码，直到取得最内层载荷。

该题的主要工作是压缩包结构识别、弱密码枚举与证据提取，因此归入 forensics，而不是需要密码数学攻击的 crypto。

## 解题过程

每层只有 100 个密码候选，最坏查询量约为

$$
3000\times100=300000
$$

次，直接穷举足够。密码必须使用 `f"{value:02d}"` 生成，才能保留 `00`～`09` 的前导零。

不需要把 3000 层文件反复写入当前目录，更不应在成功后删除原始附件。下面的脚本用 `BytesIO` 把下一层 ZIP 保存在内存中；只有全部处理完成后，才把最终载荷写入新文件。

```python
from io import BytesIO
from pathlib import Path
from zipfile import BadZipFile, ZipFile

data = Path("3000.zip").read_bytes()
depth = 0

while True:
    try:
        archive = ZipFile(BytesIO(data))
    except BadZipFile:
        break

    with archive:
        members = [item for item in archive.infolist() if not item.is_dir()]
        if len(members) != 1:
            raise ValueError(
                f"第 {depth + 1} 层应只有一个文件，实际为 {len(members)} 个"
            )

        member = members[0]
        for value in range(100):
            password = f"{value:02d}".encode()
            try:
                next_data = archive.read(member, pwd=password)
            except RuntimeError as exc:
                if "password" not in str(exc).lower():
                    raise
                continue
            else:
                print(
                    f"depth={depth + 1:04d} "
                    f"password={password.decode()} file={member.filename!r}"
                )
                break
        else:
            raise RuntimeError(f"第 {depth + 1} 层没有匹配的两位数字密码")

    data = next_data
    depth += 1

if depth != 3000:
    raise ValueError(f"预期 3000 层，实际提取 {depth} 层")

output = Path("final_payload.bin")
output.write_bytes(data)
print(f"最终载荷：{output}，大小 {len(data)} 字节")
```

脚本通过 `ZipFile` 是否还能解析当前数据来判断嵌套是否结束，并断言最终层数确实为 3000。得到 `final_payload.bin` 后，再依据文件头或 `file` 命令识别真实类型并查看其中内容。

## 方法总结

面对深层嵌套压缩包，应把“识别当前层、枚举有限密码空间、验证成员数量、读取下一层”做成确定流程。内存处理可避免超长路径、同名覆盖和误删原附件；显式检查异常与最终层数则能防止脚本在中途失败后仍给出看似成功的结果。
