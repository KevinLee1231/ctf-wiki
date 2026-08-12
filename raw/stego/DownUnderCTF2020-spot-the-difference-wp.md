# DownUnderCTF 2020 - Spot the difference!

## 题目简述

题目模拟了一份嫌疑人文件系统：`badfiles` 中有 201 张 JPEG，其中一张通过 `steghide` 藏有证据；隐藏目录 `.config/secret` 又包含 $40 \times 40=1600$ 个文本文件，用大量随机 Base64 字符串掩护真正口令。`.config/Reminder.png` 的文件头遭到破坏，修复后会给出口令中间的一小段。主要障碍是定位并提取多层隐藏载荷，因此归入 Stego。

## 解题过程

### 修复 Reminder.png

`file Reminder.png` 会误报为 ZIP。十六进制检查可以解释原因：文件前 8 字节是

```text
50 4b 03 04 0d 0a 1a 0a
```

后四字节已经与 PNG 签名一致，只有前四字节被替换成 ZIP 的 `PK\x03\x04`。把它改回 `89 50 4e 47` 后，完整 PNG 签名恢复为 `89 50 4e 47 0d 0a 1a 0a`。

```python
from pathlib import Path

data = bytearray(Path("Reminder.png").read_bytes())
assert data[:4] == b"PK\x03\x04"
data[:4] = bytes.fromhex("89504e47")
Path("Reminder-fixed.png").write_bytes(data)
```

修复后的图片只有一行文字，关键内容是：口令的中间包含 `1cmVQ`。这是可直接转写的文本线索，所以不再额外保留图片副本。

### 从 1600 个候选中找到完整口令

递归搜索该片段即可从 `.config/secret` 中筛出唯一候选：

```bash
grep -R -n '1cmVQ' .config/secret
```

命中的 Base64 字符串为：

```text
CjEyMzRJc0FTZWN1cmVQYXNzd29yZA==
```

解码结果以换行字节开头：

```bash
printf '%s' 'CjEyMzRJc0FTZWN1cmVQYXNzd29yZA==' | base64 -d | xxd
```

```text
00000000: 0a31 3233 3449 7341 5365 6375 7265 5061  .1234IsASecurePa
00000010: 7373 776f 7264                           ssword
```

去掉开头的 `0a` 后，真正的 `steghide` 口令是 `1234IsASecurePassword`。

### 批量尝试 201 张图片

弱口令 `password` 只能解出诱饵文本 “Here's a tip, try harder”。使用刚恢复的正确口令遍历全部 JPEG，并把每次输出隔离到单独目录：

```bash
mkdir -p extracted
pass='1234IsASecurePassword'

for image in badfiles/*.jpg; do
    name=$(basename "$image" .jpg)
    mkdir -p "extracted/$name"
    steghide extract -sf "$image" -p "$pass" \
        -xf "extracted/$name/payload" -f >/dev/null 2>&1 || true
done

find extracted -type f -size +0c -print -exec cat {} \;
```

只有真正的载体能通过该口令完成提取，得到：

```text
DUCTF{m0r3_th4n_M33ts_th3_ey3}
```

## 方法总结

- 核心技巧：通过 magic bytes 修复伪装文件，再用短线索筛选海量 Base64 候选，最后批量调用 `steghide` 定位唯一载体。
- 识别信号：扩展名与 magic bytes 冲突、提示图无法打开、同目录出现大量结构一致的候选文件时，应先恢复格式并寻找可用于降维搜索的稳定片段。
- 复用要点：PNG 的完整 8 字节签名必须整体核对；Base64 解码结果可能含不可见换行；批量提取时要隔离输出并检查退出状态，不能把使用弱口令得到的诱饵当成最终证据。
