# 🪐 高频率星球

## 题目简述

附件是 asciinema v2 终端录像 `asciinema_restore.rec`，录制了用户在终端中快速浏览一份约 900 KiB 的混淆 JavaScript。flag 不直接存在于录像元数据里；需要从录像的输出事件和终端控制序列中重建 `flag.js`，再用 Node.js 执行。

asciinema v2 的首行是会话元数据，后续每行是 JSON 数组：

```json
[0.90025, "o", "s"]
```

三个字段依次表示相对时间戳、输出事件类型和该时刻写入终端的字节串。

录像中的代码由 `less` 输出，夹杂清屏、擦除行尾、退格修正、shell 提示符和最后的验证命令。决定性主障碍是从已取得的终端证据中恢复文件，因此归入 `forensics`。

## 解题过程

### 提取输出事件

使用 `asciinema cat asciinema_restore.rec` 可以按录像语义输出终端字节流，但直接重放受终端可视区域和滚屏缓冲限制。更稳定的方法是逐行解析 JSON，只连接类型为 `o` 的输出字段。

官方录像前 38 行是输入 `sha256sum flag.js` 等准备过程，从第 39 行开始才是代码展示。下面的脚本先完成确定性的初步清洗：

```python
import json

lines = open("asciinema_restore.rec", encoding="utf-8").read().splitlines()
restored = ""

for line in lines[38:]:
    event = json.loads(line)
    chunk = event[-1]

    # less 状态栏从冒号和 EL（Erase in Line）开始，后面不是文件正文。
    if ":\x1b[K" in chunk:
        chunk = chunk[:chunk.index(":\x1b[K")]

    restored += (
        chunk
        .replace("\x1b[K", "")
        .replace("~\x08~", "")
        .replace("6\x086", "")
        .replace("ESC\x08\x08\x08ESC[\x08[", "")
    )

open("flag.js.raw", "w", encoding="utf-8").write(restored)
```

这些替换分别去掉 ANSI 擦除行尾序列和录像中可见的退格修正伪影。初步结果末尾仍包含离开 `less` 后的提示符、`echo "Execute flag.js ..."`、`node flag.js` 等会话噪声；将它们从最后一条 JavaScript 语句之后截断，并删除代码间由终端分页产生的空行，最后补一个换行符，保存为 `flag.js`。

### 验证恢复结果

录像开头记录了原文件的 SHA-256。恢复文件应满足：

```text
6bbbb91b7adc465fa086ec4ad453bca38beef9967800bf24d046a27b8cb70042  flag.js
```

先校验哈希，再执行：

```bash
sha256sum flag.js
node flag.js
```

哈希完全一致说明终端输出中的代码、空行和末尾换行都已精确恢复；Node.js 随后打印 flag。若脚本能执行但哈希不一致，应继续检查分页空行、退格字符和文件结尾，而不能把“语法可运行”当作完整恢复的证据。

## 方法总结

- 核心技巧：解析 asciinema v2 JSONL 输出事件，剥离 ANSI/退格/状态栏噪声，重建终端中展示的原文件。
- 识别信号：附件只有终端录像，目标文件被 `less` 快速翻页展示，录像中又给出了原文件哈希和执行提示。
- 复用要点：终端录像记录的是带控制序列的输出流，不等于最终屏幕文本。恢复时应区分文件内容、终端状态控制和 shell 会话噪声，并使用已知哈希做字节级验证。
