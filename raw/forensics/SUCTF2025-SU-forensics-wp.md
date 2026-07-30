# SU_forensics

## 题目简述

题目给出一份 VMware 虚拟机镜像。用户删除了 `.bash_history` 并执行 `history -c`，但磁盘未分配区仍可能保留文件内容。恢复出的命令继续指向已删除网页、已删除 GitHub 分支、经过涂抹的归档口令和一张自定义符号密文图，最终需要把多层“消失的数据”逐一找回。

这是一条跨磁盘、网页快照、Git 对象和图像的证据恢复链。公开网页参与了解题，但最初和持续的主障碍都是恢复失效或删除的工件，因此归入 Forensics。

## 解题过程

### 1. 从磁盘残留恢复历史命令

不要直接启动并修改唯一的原始虚拟机镜像；官方题解通过 GRUB 改密登录虽然可行，却会向磁盘写入新数据。更稳妥的做法是先复制镜像或创建只读快照，再挂载文件系统或对恢复出的 `/dev/sda1` 做字节搜索。

以 `sudo reboot` 为锚点搜索可打印上下文：

```bash
sudo grep -a -B 50 -A 50 'sudo reboot' /dev/sda1 \
  | tr -cd '\11\12\15\40-\176' \
  > result.txt
```

也可用文件恢复工具搜索已删除的 `.bash_history`。恢复出的关键命令为：

```bash
echo "My secret has disappeared from this space and time, and you will never be able to find it."
curl -s -o /dev/null https://www.cnblogs.com/cuisha12138/p/18631364
sudo reboot
```

原博客已经返回 404，但 [Wayback Machine 在 2024-12-25 保存的快照](https://web.archive.org/web/20241225122922/https://www.cnblogs.com/cuisha12138/p/18631364)仍能访问。快照只有一张截图，却给出后续证据：

- 仓库原地址为 `github.com/testtttsu/homework/tree/secret`；
- `homework.py` 把 `lost_flag.txt` 当十六进制数据还原为 `secret.zip`；
- 归档口令在截图中被半透明画笔涂黑；
- 删除分支上的提交短哈希以 `a4b` 开头。

### 2. 找回删除分支中的归档

分支删除后，只要底层 Git 对象仍保留并可寻址，提交 URL 仍可能访问。枚举最后一位可定位到短哈希 `a4be`；完整提交为：

```text
a4be9c81ae540340f3e208dc9b1ee109ea50305c
```

[该历史提交](https://github.com/testtttsu/homework/commit/a4be9c81ae540340f3e208dc9b1ee109ea50305c)新增了 `lost_flag.txt`。GitHub 的仓库 Activity/Compare 记录也能直接找到同一提交，不必盲枚举整个对象空间。

把 `lost_flag.txt` 的十六进制还原为 ZIP：

```python
from pathlib import Path

hex_data = Path("lost_flag.txt").read_text(encoding="utf-8").strip()
Path("secret.zip").write_bytes(bytes.fromhex(hex_data))
```

回看 Wayback 截图并提高曲线/对比度，可以恢复半透明涂抹下的口令：

```text
2phxMo8iUE2bAVvdsBwZ
```

使用该口令解压：

```bash
7z x -p2phxMo8iUE2bAVvdsBwZ secret.zip
```

得到原始手写符号图：

![按 69×12 网格排列的手写替换符号密文，保留重复字形与原始行列关系](SUCTF2025-SU-forensics-wp/handwritten-symbol-cipher.png)

### 3. 把手写符号转为替换密文

原图宽高可按每格 $138\times108$ 像素切成 $69\times12$ 个单元。符号本身不对应任何现成字母表，只要把相同图形聚为同一类，并给每一类分配任意占位字符即可。由于同一符号是重复粘贴，既可比较像素哈希，也可用二值图差异做模板匹配。

识别后共有约 27 类符号。频率最高的一类出现 104 次，远高于其他类别，应先视为空格。保留词边界后，对剩余符号做普通单表替换分析，可以恢复十二行英文：

```text
a quick zephyr blow vexing daft jim
fred specialized in the job of making very qabalistic wax toys
six frenzied kings vowed to abolish my quite pitiful jousts
may jo equal my foolish record by solving six puzzles a week
harry is jogging quickly which axed zen monks with abundant vapor
dumpy kibitzer jingles as quixotic overflows
nymph sing for quick jigs vex bud in zestful twilight
simple fox held quartz duck just by wing
strong brick quiz whangs jumpy fox vividly
ghosts in memory picks up quartz and valuable onyx jewels
pensive wizards make toxic brew for the evil qatari king and wry jack
all outdated query asked by five watch experts amazed the judge
```

### 4. 提取每行“消失”的字母

这些句子接近 pangram，但每行恰好缺一个英文字母。逐行计算字母表差集：

```python
import string

alphabet = set(string.ascii_lowercase)

for line in plaintext.splitlines():
    present = set(line.lower()) & alphabet
    missing = alphabet - present
    assert len(missing) == 1
    print(next(iter(missing)), end="")
```

十二行依次缺少：

```text
s u c t f h a v e f u n
```

按题目要求大写并补上 flag 格式：

```text
SUCTF{HAVEFUN}
```

## 方法总结

整条链的共同主题是“删除不等于立即不可恢复”：文件系统删除只移除元数据引用，网页 404 仍可能有历史快照，Git 分支删除不必然立即清除提交对象，半透明涂抹也没有真正覆盖像素。最后的手写字符不是要识别某种幻想文字，而是先做同形聚类，再按单表替换和词频恢复文本；十二个 lipogram 句子中缺失的字母才组成 flag。

取证时应尽量在副本上只读操作，并记录磁盘偏移、快照时间、提交哈希、归档哈希和图像尺寸。这样每次“找回”都有可追溯的证据，而不是只保留最终字符串。
