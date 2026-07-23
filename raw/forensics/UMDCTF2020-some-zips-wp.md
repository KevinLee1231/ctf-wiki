# SomeZips

## 题目简述

附件是层层嵌套的 AES 加密 ZIP，共约 384 层。每一层同时放入多个候选文件名，正确密码来自当前层中的文件名；解开后继续处理下一层，直到出现最终文本。

## 解题过程

先列出当前 ZIP 的成员而不解压，根据题目构造规律选择密码候选。多数层中，正确密码就是名称最长、体积也最显眼的那个成员名。对候选逐一尝试，成功后只保留下一层 ZIP 并循环：

```python
from pathlib import Path
import pyzipper

current = Path("somezips.zip")
work = Path("layers")
work.mkdir(exist_ok=True)

for depth in range(1000):
    layer = work / f"{depth:03d}"
    layer.mkdir()

    with pyzipper.AESZipFile(current) as archive:
        names = archive.namelist()
        probe = next(name for name in names if not name.endswith("/"))
        candidates = sorted(
            (Path(name).name for name in names),
            key=len,
            reverse=True,
        )

        for password in candidates:
            try:
                archive.read(probe, pwd=password.encode())
                break
            except (RuntimeError, ValueError):
                continue
        else:
            raise RuntimeError(f"第 {depth} 层没有可用密码")

        archive.extractall(layer, pwd=password.encode())

    next_archives = list(layer.rglob("*.zip"))
    if not next_archives:
        for result in layer.rglob("*"):
            if result.is_file():
                print(result.read_text(errors="replace"))
        break
    current = max(next_archives, key=lambda path: path.stat().st_size)
```

每层使用独立目录，避免旧 ZIP 被误选。解到末层后得到：

```text
UMDCTF-xkcd1319
```

该题使用了不带花括号的非标准格式，字符串的 SHA-256 与 README 给出的哈希一致。

## 方法总结

深层嵌套压缩包的难点是状态管理，而不是手工重复输入。自动化时要区分“列出的候选名”“本轮新解出的文件”和“上一轮遗留文件”，并在结束后用题目哈希核验非标准 flag。
