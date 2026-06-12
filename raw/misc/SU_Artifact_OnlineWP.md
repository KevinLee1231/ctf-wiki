# SU_Artifact_Online

## 题目简述
题目提供一段符文文本和一个可交互 artifact。提示说明需要构造命令寻找当前目录之外的 secret；附件 `something mysterious.txt` 是符文编码长文本，它的价值在于提供符文到明文的替换映射样本，而不是需要全文写入 WP。

远程 artifact 本质上是“5x5 六面魔方翻转 + 激活面选字符 + 执行 Linux 命令”。每次连接后魔方都会随机打乱，并且有约 40 秒时间限制，因此解题目标不是手动试字符，而是先从符文文本恢复字符映射，再本地模拟六面 cube 的 `R/C/F` 旋转和激活面取字规则，搜索出可执行命令后一次性回放操作序列。

## 解题过程
题目提示里最关键的一句是:

```text
hint: Try to craft some commands to find the secret outside the current directory.
```

这句基本已经把方向点明了:
- 这不是单纯猜一个“魔法单词”
- artifact 最终是可以执行命令的
- flag 不在当前目录，而是在上一层目录

前期分析

1.`something mysterious.txt` 并不是随机符文

先对 `something mysterious.txt` 做替换分析，可以发现它对应的是 Robert A. Heinlein 的 `All You Zombies` 片段。

这一步的意义有两个:

- 能拿到一套比较完整的 plain -> rune 映射

- 说明 artifact 输入的“咒语”很可能本质上就是某种符文编码字符串

我本地整理出来并用于脚本的主要映射如下:

```
a -> ᚠ b -> ᚢ c -> ᚦ d -> ᚨ e -> ᚱ f -> ᚲ
g -> ᚷ h -> ᚹ i -> ᚺ k -> ᛁ l -> ᛃ m -> ᛇ
n -> ᛈ o -> ᛉ p -> ᛋ r -> ᛒ s -> ᛖ t -> ᛗ
u -> ᛚ v -> ᛜ w -> ᛟ x -> ᛞ y -> ᛣ
space -> ᛨ . -> ᛧ , -> ᛥ ; -> ᛦ
```

2. artifact 本质是“转魔方 + 选字符 + 执行命令”

连上靶机之后可以看到一个 5x5 的 cube 面板。题目实际上分成两层:

- Twist 模式：通过 `R1` 到 `R5`、`C1` 到 `C5`、`F1` 到 `F5` 转动 5x5 cube，命令后加 `'` 表示逆操作。
- Activate 模式：锁定当前正前面作为激活面，从横向移动开始选字符；每确认一个字符后，下一步移动方向在横向/纵向之间交替，最后组成命令执行。

出题人 WP 对 `R/C/F` 的语义说明很关键：

- `R` 操作正前面的行，`1` 到 `5` 对应第一行到第五行，正向时 right 面的对应行向 front 面移动。
- `C` 操作正前面的列，正向时 front 面的对应列向下翻转。
- `F` 操作右侧面的列，正向时 right 面的对应列向下翻转。

所以这题的核心不是手玩，而是:

1. 自动提取六个面

2. 本地模拟所有 twist

3. 搜索目标字符串能否在某个面上被合法取出

4. 自动回放整条操作链

自动化思路

1. 用 pwntools 处理 PoW 和交互

脚本用 pwntools.remote(..., ssl=True) 连接，然后自动:

- 收 banner

- 提取 PoW 前缀

- 爆破 sha256(prefix + S) 的前缀匹配

- 发送答案

- 进入主菜单

2. 提取六个面

做法是:

- 先进入 Twist

- 读取当前 [Front] 和 [Right]

- 用预设的整面旋转序列把 B/L/U/D 依次转到可见位置

- 本地同步记录 F/R/B/L/U/D 六个面

也可以按出题人 WP 的方式，用 `R` 和 `C` 对每行/列做正反翻转探测：每个操作连续四次会回到原状态，因此可以通过 4-cycle 观察每个位置的闭合环，从而还原六面字符和移动置换。

3. 本地建模 + 搜索

CubeMatrix / FlatCubeModel 来模拟:

- 行旋转 R1~R5

- 列旋转 C1~C5

- 前后层旋转 F1~F5

再把每种 move 编译成 permutation，搜索时直接对 bytes state 做置换。

如果目标命令较长，单纯 BFS 很容易爆炸。出题人给出的思路是先固定已经能放到正面的字符，后续移动尽量不破坏已固定布局；当单步操作无法继续时，可以引入交换子：

```text
[m1, m2] = m1 · m2 · m1' · m2'
```

交换子通常只影响少量位置，适合在保留已有布局的情况下继续移动新字符。当前脚本采用的是等价的自动搜索策略：把所有 move 编译成置换后，用 BFS/beam search 寻找能在某个面上按激活规则读出目标命令的状态。

4.关键

核心有两点:

- 更好的 beam 评分：增加 `activation_frontier_stats`，在 `best_face_score()` 里不仅看缺字数，还看可延伸前缀和 frontier 大小。

- 分层加压搜索
  - `solve_target()` 会自动尝试 `beam search depth=max_depth width=beam_width`
  - 再尝试 `beam search depth=max_depth width=beam_width*2`
  - 再尝试 `beam search depth=max_depth+1`

这样长命令不会一上来就死在单一参数上。

也就是:

- 短命令继续 BFS

- 长命令走自适应 beam search

确认“命令执行”这条路是对的

前面先用短命令验证整条利用链没有走偏。

`pwd`

成功输出:

```
/home/ctf
```

说明:

- 当前目录是 /home/ctf

2.`find ..`

成功看到:

```
..
../flag
../ctf
../ctf/.bash_logout
../ctf/.bashrc
../ctf/.profile
../ctf/server.py
```

说明:
- flag 确实在上一层

- 当前目录对应的是 /home/ctf

- 上一层就是 /home

到这里其实题目就已经被拆成一句话了:

```
只要想办法执行一条“进入上一层再读 flag”的命令，就结束。
```

真正的突破点

关键思路其实非常简单:

- 不再执着于 cat ../flag

- 直接用 shell 串联命令

- 避开 /

由于符文映射里没有 `/`，所以 `cat ../flag` 不是最稳路径。先执行 `cd ..` 进入上一层，再用 `;` 拼接读文件命令即可。出题人提到 `cd ..;cat flag` 也可能成功，但为保证可执行性，源码里预设了 `cd ..;nl flag` 路径，因此 `nl` 更稳定。最后成功的命令是:

```
cd ..;nl flag
```

```
--- activating ---

1 SUCTFᚪTh1s_i5_@_Cub3_bu7_n0t_5ome7hing_u_pl4yᚫ
```

```python
#!/usr/bin/env python3
import argparse
import hashlib
import re
import sys
import time
from collections import Counter
from collections import deque
from itertools import count
from operator import itemgetter
from typing import Dict, Iterable, List, Optional, Sequence, Tuple

from pwn import context, remote

HOST = "pwn-d1533b91d4.adworld.xctf.org.cn"
PORT = 9999
SIZE = 5
RUNE_RE = re.compile(r"[\u16A0-\u16FF]")

Y_MOVES = ["R1", "R2", "R3", "R4", "R5"]
YI_MOVES = ["R5'", "R4'", "R3'", "R2'", "R1'"]
X_MOVES = ["C1", "C2", "C3", "C4", "C5"]
XI_MOVES = ["C5'", "C4'", "C3'", "C2'", "C1'"]

FACE_TO_FRONT = {
"F": [],
"R": Y_MOVES[:],
"B": Y_MOVES[:] + Y_MOVES[:],
"L": Y_MOVES[:] + Y_MOVES[:] + Y_MOVES[:],
"D": X_MOVES[:],
"U": XI_MOVES[:],
}

# Candidate rune-words derived from All You Zombies themes.
CANDIDATE_RUNES: Dict[str, List[str]] = {
"all": ["ᚨᛚᛚ", "ᚨᛚᚲ"],
"you": ["ᛦᛟᚢ", "ᛧᛟᚢ", "ᛨᛟᚢ", "ᛃᛟᚢ"],
"boy": ["ᛒᛟᛃ", "ᛒᛟᛦ", "ᛒᛟᛧ", "ᛒᛟᛨ"],
"byron": ["ᛒᛦᚱᛟᚾ", "ᛒᛧᚱᛟᚾ", "ᛒᛨᚱᛟᚾ", "ᛒᛃᚱᛟᚾ"],
"bomb": ["ᛒᛟᛗᛒ"],
"bomber": ["ᛒᛟᛗᛒᛖᚱ"],

"bar": ["ᛒᚨᚱ"],
"bartender": ["ᛒᚨᚱᛏᛖᚾᛞᛖᚱ"],
"baby": ["ᛒᚨᛒᛦ", "ᛒᚨᛒᛧ", "ᛒᚨᛒᛨ", "ᛒᚨᛒᛃ"],
"birth": ["ᛒᛁᚱᚦ"],
"bottle": ["ᛒᛟᛏᛏᛚᛖ"],
"child": ["ᚲᚺᛁᛚᛞ", "ᛤᚺᛁᛚᛞ"],
"circle": ["ᚲᛁᚱᚲᛚᛖ", "ᛤᛁᚱᛤᛚᛖ", "ᚲᛁᚱᛤᛚᛖ", "ᛤᛁᚱᚲᛚᛖ"],
"causal": ["ᚲᚨᚢᛋᚨᛚ", "ᛤᚨᚢᛋᚨᛚ"],
"cycle": ["ᚲᛦᚲᛚᛖ", "ᚲᛧᚲᛚᛖ", "ᚲᛨᚲᛚᛖ", "ᚲᛃᚲᛚᛖ"],
"daughter": ["ᛞᚨᚢᚷᚺᛏᛖᚱ"],
"self": ["ᛋᛖᛚᚠ"],
"jane": ["ᛃᚨᚾᛖ"],
"janey": ["ᛃᚨᚾᛖᛃ", "ᛃᚨᚾᛖᛦ", "ᛃᚨᚾᛖᛧ", "ᛃᚨᚾᛖᛨ"],
"ring": ["ᚱᛁᛜ"],
"snake": ["ᛋᚾᚨᚲᛖ", "ᛋᚾᚨᛤᛖ"],
"nasty": ["ᚾᚨᛋᛏᛃ", "ᚾᚨᛋᛏᛦ", "ᚾᚨᛋᛏᛧ", "ᚾᚨᛋᛏᛨ"],
"needless_risks": ["ᚾᛖᛖᛞᛚᛖᛋᛋᛨᚱᛁᛋᚲᛋ", "ᚾᛖᛖᛞᛚᛖᛋᛋᛨᚱᛁᛋᛤᛋ"],
"touchy": ["ᛏᛟᚢᚲᚺᛃ", "ᛏᛟᚢᚲᚺᛦ", "ᛏᛟᚢᚲᚺᛧ", "ᛏᛟᚢᚲᚺᛨ"],
"touchy_temper": [
"ᛏᛟᚢᚲᚺᛃᛨᛏᛖᛗᛈᛖᚱ",
"ᛏᛟᚢᚲᚺᛦᛨᛏᛖᛗᛈᛖᚱ",
"ᛏᛟᚢᚲᚺᛧᛨᛏᛖᛗᛈᛖᚱ",
"ᛏᛟᚢᚲᚺᛨᛨᛏᛖᛗᛈᛖᚱ",
],
"racket": ["ᚱᚨᚲᚲᛖᛏ", "ᚱᚨᛤᛤᛖᛏ", "ᚱᚨᚲᛤᛖᛏ", "ᚱᚨᛤᚲᛖᛏ"],
"double_shot": ["ᛞᛟᚢᛒᛚᛖᛨᛋᚺᛟᛏ"],
"recruit": ["ᚱᛖᚲᚱᚢᛁᛏ", "ᚱᛖᛤᚱᚢᛁᛏ"],
"sap": ["ᛋᚨᛈ"],
"swish": ["ᛋᚹᛁᛋᚺ", "ᛉᚹᛁᛋᚺ", "ᛋᚹᛁᛉᚺ", "ᛉᚹᛁᛉᚺ"],
"critical": ["ᚲᚱᛁᛏᛁᚲᚨᛚ", "ᛤᚱᛁᛏᛁᛤᚨᛚ", "ᚲᚱᛁᛏᛁᛤᚨᛚ", "ᛤᚱᛁᛏᛁᚲᚨᛚ"],
"tail": ["ᛏᚨᛁᛚ"],
"own_tail": ["ᛟᚹᚾᛨᛏᚨᛁᛚ"],
"its_own_tail": ["ᛁᛏᛋᛨᛟᚹᚾᛨᛏᚨᛁᛚ"],
"eats_its_own_tail": ["ᛖᚨᛏᛋᛨᛁᛏᛋᛨᛟᚹᚾᛨᛏᚨᛁᛚ"],
"temporal": ["ᛏᛖᛗᛈᛟᚱᚨᛚ"],
"manipulation": ["ᛗᚨᚾᛁᛈᚢᛚᚨᛏᛁᛟᚾ"],
"temporal_manipulation": ["ᛏᛖᛗᛈᛟᚱᚨᛚᛨᛗᚨᚾᛁᛈᚢᛚᚨᛏᛁᛟᚾ"],
"temporal_bureau": ["ᛏᛖᛗᛈᛟᚱᚨᛚᛨᛒᚢᚱᛖᚨᚢ"],
"ever": ["ᛖᚢᛖᚱ"],
"and_ever": ["ᚨᚾᛞᛨᛖᚢᛖᚱ"],
"forever": ["ᚠᛟᚱᛖᚢᛖᚱ"],
"forever_and_ever": ["ᚠᛟᚱᛖᚢᛖᚱᛨᚨᚾᛞᛨᛖᚢᛖᚱ"],
"snake_that_eats_its_own_tail": [
"ᛋᚾᚨᚲᛖᛨᚦᚨᛏᛨᛖᚨᛏᛋᛨᛁᛏᛋᛨᛟᚹᚾᛨᛏᚨᛁᛚ",
"ᛋᚾᚨᛤᛖᛨᚦᚨᛏᛨᛖᚨᛏᛋᛨᛁᛏᛋᛨᛟᚹᚾᛨᛏᚨᛁᛚ",
],
"the_snake_that_eats_its_own_tail": [

"ᚦᛖᛨᛋᚾᚨᚲᛖᛨᚦᚨᛏᛨᛖᚨᛏᛋᛨᛁᛏᛋᛨᛟᚹᚾᛨᛏᚨᛁᛚ",
"ᚦᛖᛨᛋᚾᚨᛤᛖᛨᚦᚨᛏᛨᛖᚨᛏᛋᛨᛁᛏᛋᛨᛟᚹᚾᛨᛏᚨᛁᛚ",
],
"all_you_zombies": [
"ᚨᛚᛚᛨᛦᛟᚢᛨᛋᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛨᛧᛟᚢᛨᛋᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛨᛨᛟᚢᛨᛋᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛨᛃᛟᚢᛨᛋᛟᛗᛒᛁᛖᛋ",
],
"allyouzombies": [
"ᚨᛚᛚᛦᛟᚢᛉᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛧᛟᚢᛉᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛨᛟᚢᛉᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛃᛟᚢᛉᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛦᛟᚢᛋᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛧᛟᚢᛋᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛨᛟᚢᛋᛟᛗᛒᛁᛖᛋ",
"ᚨᛚᛚᛃᛟᚢᛋᛟᛗᛒᛁᛖᛋ",
],
"my_own_grandpa": [
"ᛗᛦᛨᛟᚹᚾᛨᚷᚱᚨᚾᛞᛈᚨ",
"ᛗᛧᛨᛟᚹᚾᛨᚷᚱᚨᚾᛞᛈᚨ",
"ᛗᛨᛨᛟᚹᚾᛨᚷᚱᚨᚾᛞᛈᚨ",
"ᛗᛃᛨᛟᚹᚾᛨᚷᚱᚨᚾᛞᛈᚨ",
],
"own_grandpa": ["ᛟᚹᚾᛨᚷᚱᚨᚾᛞᛈᚨ"],
"zombie": ["ᛋᛟᛗᛒᛁᛖ"],
"zombies": ["ᛋᛟᛗᛒᛁᛖᛋ"],
"unmarried_mother": ["ᚢᚾᛗᚨᚱᚱᛁᛖᛞᛨᛗᛟᚦᛖᚱ"],
"time": ["ᛏᛁᛗᛖ"],
"mother": ["ᛗᛟᚦᛖᚱ"],
"parent": ["ᛈᚨᚱᛖᚾᛏ"],
"pops": ["ᛈᛟᛈᛋ", "ᛈᛟᛈᛉ"],
"father": ["ᚠᚨᚦᛖᚱ"],
"fate": ["ᚠᚨᛏᛖ"],
"fizzle": ["ᚠᛁᛋᛋᛚᛖ"],
"fizzle_bomber": ["ᚠᛁᛋᛋᛚᛖᛨᛒᛟᛗᛒᛖᚱ"],
"heinlein": ["ᚺᛖᛁᚾᛚᛖᛁᚾ"],
"know": ["ᚲᚾᛟᚹ", "ᛤᚾᛟᚹ"],
"grandpa": ["ᚷᚱᚨᚾᛞᛈᚨ"],
"janus": ["ᛃᚨᚾᚢᛋ"],
"loop": ["ᛚᛟᛟᛈ"],
"paradox": ["ᛈᚨᚱᚨᛞᛟᚲ", "ᛈᚨᚱᚨᛞᛟᛤ"],
"machine": ["ᛗᚨᚲᚺᛁᚾᛖ", "ᛗᚨᛤᚺᛁᚾᛖ"],
"wyrd": ["ᚹᛦᚱᛞ", "ᚹᛧᚱᛞ", "ᚹᛨᚱᛞ", "ᚹᛃᚱᛞ"],
"war": ["ᚹᚨᚱ"],
"history": ["ᚺᛁᛋᛏᛟᚱᛃ", "ᚺᛁᛋᛏᛟᚱᛦ", "ᚺᛁᛋᛏᛟᚱᛧ", "ᚺᛁᛋᛏᛟᚱᛨ"],

"orphanage": ["ᛟᚱᛈᚺᚨᚾᚨᚷᛖ"],
"scar": ["ᛋᚲᚨᚱ", "ᛋᛤᚨᚱ"],
"space": ["ᛋᛈᚨᚲᛖ", "ᛋᛈᚨᛤᛖ"],
"spell": ["ᛋᛈᛖᛚᛚ"],
"rune": ["ᚱᚢᚾᛖ"],
"runes": ["ᚱᚢᚾᛖᛋ", "ᚱᚢᚾᛖᛉ"],
"twist": ["ᛏᚹᛁᛋᛏ"],
"activate": ["ᚨᚲᛏᛁᚢᚨᛏᛖ", "ᚨᛤᛏᛁᚢᚨᛏᛖ"],
"verify": ["ᚢᛖᚱᛁᚠᛦ", "ᚢᛖᚱᛁᚠᛧ", "ᚢᛖᚱᛁᚠᛨ", "ᚢᛖᚱᛁᚠᛃ"],
"corps": ["ᚲᛟᚱᛈᛋ", "ᛤᛟᚱᛈᛋ"],
"clinic": ["ᚲᛚᛁᚾᛁᚲ", "ᛤᛚᛁᚾᛁᛤ", "ᚲᛚᛁᚾᛁᛤ", "ᛤᛚᛁᚾᛁᚲ"],
"orphan": ["ᛟᚱᛈᚺᚨᚾ"],
"agent": ["ᚨᚷᛖᚾᛏ"],
"artifact": ["ᚨᚱᛏᛁᚠᚨᚲᛏ", "ᚨᚱᛏᛁᚠᚨᛤᛏ"],
"bureau": ["ᛒᚢᚱᛖᚨᚢ"],
"bootstrap": ["ᛒᛟᛟᛏᛋᛏᚱᚨᛈ"],
"barkeep": ["ᛒᚨᚱᚲᛖᛖᛈ", "ᛒᚨᚱᛤᛖᛖᛈ"],
"came": ["ᚲᚨᛗᛖ", "ᛤᚨᛗᛖ"],
"came_from": ["ᚲᚨᛗᛖᛨᚠᚱᛟᛗ", "ᛤᚨᛗᛖᛨᚠᚱᛟᛗ"],
"word": ["ᚹᛟᚱᛞ"],
"confession": ["ᚲᛟᚾᚠᛖᛋᛋᛁᛟᚾ", "ᛤᛟᚾᚠᛖᛋᛋᛁᛟᚾ"],
"confession_stories": ["ᚲᛟᚾᚠᛖᛋᛋᛁᛟᚾᛨᛋᛏᛟᚱᛁᛖᛋ", "ᛤᛟᚾᚠᛖᛋᛋᛁᛟᚾᛨᛋᛏᛟᚱᛁᛖᛋ"],
"four_cents_a_word": ["ᚠᛟᚢᚱᛨᚲᛖᚾᛏᛋᛨᚨᛨᚹᛟᚱᛞ", "ᚠᛟᚢᚱᛨᛤᛖᚾᛏᛋᛨᚨᛨᚹᛟᚱᛞ"],
"old_underwear": ["ᛟᛚᛞᛨᚢᚾᛞᛖᚱᚹᛖᚨᚱ"],
"secret": ["ᛋᛖᚲᚱᛖᛏ", "ᛋᛖᛤᚱᛖᛏ"],
"stories": ["ᛋᛏᛟᚱᛁᛖᛋ"],
"truth": ["ᛏᚱᚢᛏᚺ"],
"underwear": ["ᚢᚾᛞᛖᚱᚹᛖᚨᚱ"],
"where": ["ᚹᚺᛖᚱᛖ"],
"where_i_came_from": ["ᚹᚺᛖᚱᛖᛨᛁᛨᚲᚨᛗᛖᛨᚠᚱᛟᛗ", "ᚹᚺᛖᚱᛖᛨᛁᛨᛤᚨᛗᛖᛨᚠᚱᛟᛗ"],
"from": ["ᚠᚱᛟᛗ"],
"ouroboros": ["ᛟᚢᚱᛟᛒᛟᚱᛟᛋ"],
}

STORY_CIPHER_PLAIN_TO_RUNE = {
"a": "ᚠ",
"b": "ᚢ",
"c": "ᚦ",
"d": "ᚨ",
"e": "ᚱ",
"f": "ᚲ",
"g": "ᚷ",
"h": "ᚹ",
"i": "ᚺ",
"k": "ᛁ",
"l": "ᛃ",
"m": "ᛇ",

"n": "ᛈ",
"o": "ᛉ",
"p": "ᛋ",
"r": "ᛒ",
"s": "ᛖ",
"t": "ᛗ",
"u": "ᛚ",
"v": "ᛜ",
"w": "ᛟ",
"x": "ᛞ",
"y": "ᛣ",
" ": "ᛨ",
"'": "ᚯ",
"-": "ᚬ",
}

STORY_CIPHER_EXTRA_PLAIN_TO_RUNE = {
".": "ᛧ",
",": "ᛥ",
";": "ᛦ",
'"': "ᚭ",
"?": "ᛩ",
}

COMMAND_PLAIN_TO_RUNE = {
**STORY_CIPHER_PLAIN_TO_RUNE,
**STORY_CIPHER_EXTRA_PLAIN_TO_RUNE,
}

COMMAND_RUNE_TO_PLAIN = {rune: plain for plain, rune in
COMMAND_PLAIN_TO_RUNE.items()}

STANDARD_RUNE_HINTS = {
"ᚠ": "a/f",
"ᚢ": "b/u/v",
"ᚦ": "c/th",
"ᚨ": "d/a",
"ᚱ": "e/r",
"ᚲ": "f/c/k/q",
"ᚷ": "g",
"ᚹ": "h/w",
"ᚺ": "i/h",
"ᛁ": "k/i",
"ᛃ": "l/j/y",
"ᛇ": "m",
"ᛈ": "n/p",
"ᛉ": "o/s/x/z",

"ᛋ": "p/s/z",
"ᛒ": "r/b",
"ᛖ": "s/e",
"ᛗ": "t/m",
"ᛚ": "u/l",
"ᛜ": "v/ng",
"ᛟ": "w/o",
"ᛞ": "x/d",
"ᛣ": "y",
"ᛤ": "c/k/q",
"ᚭ": '"',
"ᚬ": "-",
"ᚯ": "'",
"ᛥ": ",",
"ᛦ": ";/y",
"ᛧ": "./y",
"ᛨ": "space/y",
"ᛩ": "?",
}

def encode_story_cipher(text: str) -> Optional[str]:
out: List[str] = []
for ch in text.lower():
rune = STORY_CIPHER_PLAIN_TO_RUNE.get(ch)
if rune is None:
return None
out.append(rune)
return "".join(out)

def encode_command_text(text: str) -> Optional[str]:
out: List[str] = []
for ch in text.lower():
rune = COMMAND_PLAIN_TO_RUNE.get(ch)
if rune is None:
return None
out.append(rune)
return "".join(out)

def decode_command_output(text: str) -> str:
return "".join(COMMAND_RUNE_TO_PLAIN.get(ch, ch) for ch in text)

def describe_charset(faces: Dict[str, List[List[str]]]) -> List[str]:
observed = Counter()
for face in faces.values():
for row in face:
observed.update(row)

lines = []
for rune, count in sorted(observed.items(), key=lambda item: item[0]):
plain = COMMAND_RUNE_TO_PLAIN.get(rune)
if plain is not None:
label = repr(plain) if plain == " " else plain
lines.append(f"{rune} -> command {label} ({count})")
else:
hint = STANDARD_RUNE_HINTS.get(rune, "?")
lines.append(f"{rune} -> extra/standard {hint} ({count})")
return lines

def standard_variants(text: str, limit: int = 256) -> List[str]:
char_map = {
"a": ["ᚨ"],
"b": ["ᛒ"],
"c": ["ᚲ", "ᛤ"],
"d": ["ᛞ"],
"e": ["ᛖ"],
"f": ["ᚠ"],
"g": ["ᚷ"],
"h": ["ᚺ"],
"i": ["ᛁ"],
"j": ["ᛃ"],
"k": ["ᚲ"],
"l": ["ᛚ"],
"m": ["ᛗ"],
"n": ["ᚾ"],
"o": ["ᛟ"],
"p": ["ᛈ"],
"q": ["ᛤ", "ᚲ"],
"r": ["ᚱ"],
"s": ["ᛋ", "ᛉ"],
"t": ["ᛏ"],
"u": ["ᚢ"],
"v": ["ᚢ", "ᚠ"],
"w": ["ᚹ"],
"x": ["ᛉ", "ᚲᛋ"],
"y": ["ᛦ", "ᛧ", "ᛨ", "ᛃ"],
"z": ["ᛉ", "ᛋ"],
" ": ["ᛨ"],
"'": ["ᚯ"],
"-": ["ᚬ"],
}

text = text.lower()
variants = [""]
i = 0

while i < len(text):
if text.startswith("th", i):
parts = ["ᚦ", "ᛏᚺ"]
i += 2
elif text.startswith("ng", i):
parts = ["ᛜ", "ᚾᚷ"]
i += 2
else:
parts = char_map.get(text[i], [])
i += 1
if not parts:
return []
variants = [prefix + part for prefix in variants for part in parts]
if len(variants) > limit:
variants = variants[:limit]
return list(dict.fromkeys(variants))

for plain in (
"time",
"mother",
"father",
"daughter",
"grandpa",
"bar",
"boy",
"jane",
"janey",
"machine",
"snake",
"ring",
"self",
"baby",
"child",
"unmarried mother",
"truth",
"secret",
"artifact",
"space",
"human",
"parent",
):
encoded = encode_story_cipher(plain)
if encoded is not None:
CANDIDATE_RUNES[f"story_{plain.replace(' ', '_')}"] = [encoded]

for plain in (
"zombie",

"zombies",
"all you zombies",
"fizzle",
"cycle",
"where",
):
key = plain.replace(" ", "_")
merged = list(dict.fromkeys(CANDIDATE_RUNES.get(key, []) +
standard_variants(plain)))
if merged:
CANDIDATE_RUNES[key] = merged

for plain in (
"seducer",
"customer",
"spaceman",
"spacemen",
"pregnant",
"human",
"race",
"outpost",
"mountains",
"predestination",
):
key = plain.replace(" ", "_")
merged = list(dict.fromkeys(CANDIDATE_RUNES.get(key, []) +
standard_variants(plain)))
if merged:
CANDIDATE_RUNES[key] = merged

def solve_pow(banner: bytes) -> bytes:
match = re.search(
rb'sha256\("([^"]+)" \+ S\)\.hexdigest\(\)\[:\d+\] == "([0-9a-f]+)"',
banner,
)
if match is None:
raise ValueError("PoW prompt not found")

prefix, target = match.groups()
target_text = target.decode()

for i in count():
suffix = str(i).encode()
if hashlib.sha256(prefix + suffix).hexdigest().startswith(target_text):
return suffix

def recv_for(io, seconds: float) -> bytes:

end = time.time() + seconds
chunks: List[bytes] = []

while time.time() < end:
try:
data = io.recv(timeout=0.025)
except EOFError:
break
if data:
chunks.append(data)

return b"".join(chunks)

def parse_visible_faces(blob: bytes) -> List[Tuple[List[str], List[str]]]:
text = blob.decode("utf-8", "replace")

for screen in reversed(text.split("\x1b[2J\x1b[H")):
rows: List[Tuple[List[str], List[str]]] = []
capture = False

for line in screen.splitlines():
if "[Front]" in line and "[Right]" in line:
capture = True
rows = []
continue

if capture and "|" in line:
runes = RUNE_RE.findall(line)
if len(runes) >= 10:
rows.append((runes[:SIZE], runes[SIZE : SIZE * 2]))
if len(rows) == SIZE:
return rows

return []

def row_major(face: Sequence[Sequence[str]]) -> List[str]:
return [cell for row in face for cell in row]

def rot_cw(face: List[List[int]]) -> List[List[int]]:
return [[face[SIZE - 1 - r][c] for r in range(SIZE)] for c in range(SIZE)]

def rot_ccw(face: List[List[int]]) -> List[List[int]]:
return [[face[r][SIZE - 1 - c] for r in range(SIZE)] for c in range(SIZE)]

def col(face: List[List[int]], j: int) -> List[int]:
return [face[r][j] for r in range(SIZE)]

def set_col(face: List[List[int]], j: int, values: Sequence[int]) -> None:
for r in range(SIZE):
face[r][j] = values[r]

class CubeMatrix:
def __init__(self, faces: Dict[str, List[List[int]]]):
self.faces = {name: [row[:] for row in face] for name, face in
faces.items()}

def row_move(self, idx: int) -> None:
f = self.faces
f["F"][idx], f["R"][idx], f["B"][idx], f["L"][idx] = (
f["R"][idx][:],
f["B"][idx][:],
f["L"][idx][:],
f["F"][idx][:],
)

if idx == 0:
f["U"] = rot_cw(f["U"])
if idx == SIZE - 1:
f["D"] = rot_ccw(f["D"])

def col_move(self, idx: int) -> None:
f = self.faces
front = col(f["F"], idx)
up = col(f["U"], idx)
back = col(f["B"], SIZE - 1 - idx)
down = col(f["D"], idx)

set_col(f["F"], idx, down)
set_col(f["D"], idx, back[::-1])
set_col(f["B"], SIZE - 1 - idx, up[::-1])
set_col(f["U"], idx, front)

if idx == 0:
f["L"] = rot_ccw(f["L"])
if idx == SIZE - 1:
f["R"] = rot_cw(f["R"])

def front_move(self, idx: int) -> None:
f = self.faces
up = f["U"][SIZE - 1 - idx][:]
right = col(f["R"], idx)
down = f["D"][idx][:]
left = col(f["L"], SIZE - 1 - idx)

set_col(f["R"], idx, up)
f["D"][idx] = right[::-1]
set_col(f["L"], SIZE - 1 - idx, down)
f["U"][SIZE - 1 - idx] = left[::-1]

if idx == 0:
f["F"] = rot_cw(f["F"])
if idx == SIZE - 1:
f["B"] = rot_ccw(f["B"])

def apply(self, move: str) -> None:
axis = move[0]
idx = int(move[1]) - 1
turns = 3 if move.endswith("'") else 1

for _ in range(turns):
if axis == "R":
self.row_move(idx)
elif axis == "C":
self.col_move(idx)
elif axis == "F":
self.front_move(idx)
else:
raise ValueError(f"Unsupported move: {move}")

class FlatCubeModel:
FACE_OFFSETS = {
"F": 0,
"R": 25,
"B": 50,
"L": 75,
"U": 100,
"D": 125,
}

MOVES = [f"{axis}{i}{suffix}" for axis in "RCF" for i in range(1, SIZE +
1) for suffix in ("", "'")]

def __init__(self):
index_faces = {}
cur = 0
for name in ("F", "R", "B", "L", "U", "D"):
face = []
for _ in range(SIZE):
row = list(range(cur, cur + SIZE))
cur += SIZE
face.append(row)

index_faces[name] = face

perms: Dict[str, Tuple[int, ...]] = {}
for move in self.MOVES:
cube = CubeMatrix(index_faces)
cube.apply(move)
perm = []
for name in ("F", "R", "B", "L", "U", "D"):
perm.extend(row_major(cube.faces[name]))
perms[move] = tuple(perm)

self._perm_getters = {move: itemgetter(*perm) for move, perm in
perms.items()}

def apply(self, state: bytes, move: str) -> bytes:
return bytes(self._perm_getters[move](state))

def face_grid(self, state: bytes, face: str) -> List[List[int]]:
off = self.FACE_OFFSETS[face]
return [list(state[off + r * SIZE : off + (r + 1) * SIZE]) for r in
range(SIZE)]

def build_state_and_lookup(faces: Dict[str, List[List[str]]], candidates:
Iterable[str]) -> Tuple[bytes, Dict[str, int], Dict[int, str]]:
symbols = set()
for face in faces.values():
for row in face:
symbols.update(row)
for candidate in candidates:
symbols.update(candidate)

ordered = sorted(symbols)
encode = {ch: idx for idx, ch in enumerate(ordered)}
decode = {idx: ch for ch, idx in encode.items()}

values: List[int] = []
for name in ("F", "R", "B", "L", "U", "D"):
for row in faces[name]:
values.extend(encode[ch] for ch in row)

return bytes(values), encode, decode

def find_activation_path(face: List[List[int]], target: bytes) ->
Optional[List[Tuple[int, int]]]:
def dfs(idx: int, r: int, c: int, vertical: bool, path: List[Tuple[int,
int]]) -> Optional[List[Tuple[int, int]]]:
if idx == len(target):

return path[:]

want = target[idx]
if vertical:
for nr in range(SIZE):
if nr != r and face[nr][c] == want:
path.append((nr, c))
found = dfs(idx + 1, nr, c, False, path)
if found is not None:
return found
path.pop()
else:
for nc in range(SIZE):
if nc != c and face[r][nc] == want:
path.append((r, nc))
found = dfs(idx + 1, r, nc, True, path)
if found is not None:
return found
path.pop()
return None

first = target[0]
for c0 in range(SIZE):
if face[0][c0] == first:
result = dfs(1, 0, c0, True, [(0, c0)])
if result is not None:
return result

return None

def longest_activation_prefix(face: List[List[int]], target: bytes) -> int:
if not target:
return 0

cache: Dict[Tuple[int, int, int, bool], int] = {}

def dfs(idx: int, r: int, c: int, vertical: bool) -> int:
key = (idx, r, c, vertical)
if key in cache:
return cache[key]

best = idx
if idx == len(target):
cache[key] = idx
return idx

want = target[idx]

if vertical:
for nr in range(SIZE):
if nr != r and face[nr][c] == want:
best = max(best, dfs(idx + 1, nr, c, False))
else:
for nc in range(SIZE):
if nc != c and face[r][nc] == want:
best = max(best, dfs(idx + 1, r, nc, True))

cache[key] = best
return best

best = 0
first = target[0]
for c0 in range(SIZE):
if face[0][c0] == first:
best = max(best, dfs(1, 0, c0, True))
return best

def activation_frontier_stats(face: List[List[int]], target: bytes) ->
Tuple[int, int, int]:
if not target:
return (0, 0, 0)

frontier = {(0, c, True) for c in range(SIZE) if face[0][c] == target[0]}
if not frontier:
return (0, 0, 0)

total_frontier = len(frontier)
prefix = 1

for idx in range(1, len(target)):
want = target[idx]
nxt = set()
for r, c, vertical in frontier:
if vertical:
for nr in range(SIZE):
if nr != r and face[nr][c] == want:
nxt.add((nr, c, False))
else:
for nc in range(SIZE):
if nc != c and face[r][nc] == want:
nxt.add((r, nc, True))
if not nxt:
break
frontier = nxt
total_frontier += len(frontier)

prefix = idx + 1

return (prefix, len(frontier), total_frontier)

def shortest_wrap(cur: int, target: int) -> Tuple[int, int]:
pos = (target - cur) % SIZE
neg = (cur - target) % SIZE
return (pos, 1) if pos <= neg else (neg, -1)

def activation_steps(path: Sequence[Tuple[int, int]]) -> List[str]:
steps: List[str] = []
cur_r, cur_c = 0, 0
horizontal = True

for i, (tr, tc) in enumerate(path):
if i == 0:
count, direction = shortest_wrap(cur_c, tc)
steps.extend(("R" if direction == 1 else "L") for _ in
range(count))
cur_c = tc
else:
if horizontal:
count, direction = shortest_wrap(cur_c, tc)
steps.extend(("R" if direction == 1 else "L") for _ in
range(count))
cur_c = tc
else:
count, direction = shortest_wrap(cur_r, tr)
steps.extend(("D" if direction == 1 else "U") for _ in
range(count))
cur_r = tr

steps.append("E")
cur_r, cur_c = tr, tc
horizontal = not horizontal

steps.append("X")
return steps

class ArtifactClient:
def __init__(self, host: str, port: int, ssl: bool = True):
context.log_level = "error"
self.io = remote(host, port, ssl=ssl)

def close(self) -> None:
self.io.close()

def recv_until(self, marker: bytes, timeout: float = 0.5) -> bytes:
try:
return self.io.recvuntil(marker, timeout=timeout)
except EOFError:
return b""

def recv_rows(self, wait: float = 0.12, retries: int = 3) ->
List[Tuple[List[str], List[str]]]:
rows: List[Tuple[List[str], List[str]]] = []
for attempt in range(retries):
rows = parse_visible_faces(recv_for(self.io, wait))
if len(rows) == SIZE:
return rows
wait = max(wait, 0.08) * 1.5
return rows

def send_menu_line(self, text: str) -> List[Tuple[List[str], List[str]]]:
self.io.sendline(text.encode())
blob = self.recv_until(b"> ", timeout=0.6)
rows = parse_visible_faces(blob)
if len(rows) == SIZE:
return rows
return self.recv_rows(0.12, retries=4)

def send_twist_moves(self, moves: Sequence[str], wait: float = 0.05) ->
List[Tuple[List[str], List[str]]]:
rows = []
for move in moves:
self.io.sendline(move.encode())
blob = self.recv_until(b"move> ", timeout=0.6)
rows = parse_visible_faces(blob)
if len(rows) != SIZE:
rows = self.recv_rows(wait, retries=4)
if not rows:
rows = self.recv_rows(0.12, retries=4)
return rows

def send_activate_steps(self, steps: Sequence[str], key_delay: float =
0.12, read_delay: float = 0.12) -> str:
keymap = {
"L": b"\x1b[D",
"R": b"\x1b[C",
"U": b"\x1b[A",
"D": b"\x1b[B",
"E": b"\r",
"X": b"x",
}

output = []
for step in steps:
self.io.send(keymap[step])
time.sleep(key_delay)
output.append(recv_for(self.io, read_delay))
output.append(recv_for(self.io, 1.0))
return b"".join(output).decode("utf-8", "replace")

def extract_faces(client: ArtifactClient) -> Dict[str, List[List[str]]]:
rows = client.recv_rows(0.8, retries=5)
if len(rows) != SIZE:
raise RuntimeError("failed to synchronize on the main menu")
rows = client.send_menu_line("1")
if len(rows) != SIZE:
raise RuntimeError("failed to capture the initial front/right faces")
faces = {
"F": [front for front, _ in rows],
"R": [right for _, right in rows],
}

rows = client.send_twist_moves(Y_MOVES)
faces["B"] = [right for _, right in rows]

rows = client.send_twist_moves(Y_MOVES)
faces["L"] = [right for _, right in rows]

client.send_twist_moves(Y_MOVES)
client.send_twist_moves(Y_MOVES)

rows = client.send_twist_moves(X_MOVES)
faces["D"] = [front for front, _ in rows]

client.send_twist_moves(XI_MOVES)
rows = client.send_twist_moves(XI_MOVES)
faces["U"] = [front for front, _ in rows]

client.send_twist_moves(X_MOVES)
for name, face in faces.items():
if len(face) != SIZE or any(len(row) != SIZE for row in face):
raise RuntimeError(f"captured an incomplete {name} face")
return faces

def search_spell(model: FlatCubeModel, state: bytes, target: bytes, max_depth:
int) -> Optional[Tuple[List[str], str, List[Tuple[int, int]]]]:
need = Counter(target)
queue = deque([(state, [])])
seen = {state}

while queue:
cur_state, path = queue.popleft()

for face in ("F", "R", "B", "L", "U", "D"):
grid = model.face_grid(cur_state, face)
flat_face = [cell for row in grid for cell in row]
have = Counter(flat_face)
if any(have[sym] < count for sym, count in need.items()):
continue
found = find_activation_path(grid, target)
if found is not None:
return path, face, found

if len(path) == max_depth:
continue

last = path[-1] if path else None
for move in model.MOVES:
if last and last[0] == move[0] and last[1] == move[1] and
last.endswith("'") != move.endswith("'"):
continue
nxt = model.apply(cur_state, move)
if nxt in seen:
continue
seen.add(nxt)
queue.append((nxt, path + [move]))

return None

def best_face_score(
model: FlatCubeModel,
state: bytes,
target: bytes,
) -> Tuple[Tuple[int, int, int, int, int], Optional[str],
Optional[List[List[int]]]]:
need = Counter(target)
best_score = (10**9, 10**9, 10**9, 10**9, 10**9)
best_face = None
best_grid = None

for face in ("F", "R", "B", "L", "U", "D"):
grid = model.face_grid(state, face)
have = Counter(cell for row in grid for cell in row)
deficit = sum(max(0, need[sym] - have[sym]) for sym in need)
prefix, live_frontier, total_frontier =
activation_frontier_stats(grid, target)

prefix_gap = len(target) - prefix
starts = sum(1 for c in range(SIZE) if grid[0][c] == target[0]) if
```

target else 0

```python
score = (deficit, prefix_gap, -total_frontier, -live_frontier, -starts)
if score < best_score:
best_score = score
best_face = face
best_grid = grid

return best_score, best_face, best_grid

def search_spell_beam(
model: FlatCubeModel,
state: bytes,
target: bytes,
max_depth: int,
beam_width: int = 220,
) -> Optional[Tuple[List[str], str, List[Tuple[int, int]]]]:
initial_score, _, _ = best_face_score(model, state, target)
beam: List[Tuple[Tuple[int, int], Tuple[str, ...], bytes]] =
[(initial_score, tuple(), state)]
seen = {state}

for depth in range(max_depth + 1):
beam.sort(key=itemgetter(0))

for score, path, cur_state in beam:
for face in ("F", "R", "B", "L", "U", "D"):
grid = model.face_grid(cur_state, face)
found = find_activation_path(grid, target)
if found is not None:
return list(path), face, found

if depth == max_depth:
return None

next_beam: List[Tuple[Tuple[int, int], Tuple[str, ...], bytes]] = []
for score, path, cur_state in beam[:beam_width]:
last = path[-1] if path else None
for move in model.MOVES:
if last and last[0] == move[0] and last[1] == move[1] and
last.endswith("'") != move.endswith("'"):
continue
nxt = model.apply(cur_state, move)
if nxt in seen:
continue
seen.add(nxt)

next_score, _, _ = best_face_score(model, nxt, target)
next_beam.append((next_score, path + (move,), nxt))

next_beam.sort(key=itemgetter(0))
beam = next_beam[: beam_width * 4]

return None

def solve_target(
model: FlatCubeModel,
state: bytes,
target: bytes,
max_depth: int,
beam_width: int,
) -> Optional[Tuple[List[str], str, List[Tuple[int, int]]]]:
if len(target) <= 8:
return search_spell(model, state, target, max_depth)

beam_plan = [
(max_depth, beam_width),
(max_depth, beam_width * 2),
(max_depth + 1, beam_width * 2),
(max_depth + 1, beam_width * 4),
]
seen_configs = set()
for depth, width in beam_plan:
config = (depth, width)
if config in seen_configs:
continue
seen_configs.add(config)
print(f"beam search depth={depth} width={width}")
solution = search_spell_beam(model, state, target, depth,
beam_width=width)
if solution is not None:
return solution
return None

def connect_and_extract(host: str, port: int, retries: int = 3) ->
Tuple[ArtifactClient, Dict[str, List[List[str]]]]:
last_error: Optional[BaseException] = None

for attempt in range(1, retries + 1):
client = ArtifactClient(host, port, ssl=True)
try:
banner = b""
for _ in range(10):
banner += recv_for(client.io, 0.6)

if b"sha256(" in banner and b"S: " in banner:
break
if b"sha256(" not in banner or b"S: " not in banner:
raise RuntimeError(f"PoW prompt not found on attempt
{attempt}")

client.io.sendline(solve_pow(banner))
faces = extract_faces(client)
return client, faces
except Exception as exc:
last_error = exc
try:
client.close()
except Exception:
pass
time.sleep(0.2 * attempt)

assert last_error is not None
raise last_error

def run_target_variants(
target_name: str,
variants: Sequence[str],
max_depth: int,
key_delay: float,
host: str,
port: int,
beam_width: int,
decode_output: bool = False,
) -> bool:
client, faces = connect_and_extract(host, port)
try:
print("extracted faces")

state, encode, decode = build_state_and_lookup(faces, variants)
model = FlatCubeModel()

for variant in variants:
target = bytes(encode[ch] for ch in variant)
t1 = time.time()
solution = solve_target(model, state, target, max_depth,
beam_width)
elapsed = time.time() - t1
print(f"search {target_name}/{variant!r} took {elapsed:.2f}s")
if solution is None:
continue

move_path, face_name, activation_path_on_face = solution
print("solution", move_path, "face", face_name, "path",
activation_path_on_face)

all_moves = move_path + FACE_TO_FRONT[face_name]
client.send_twist_moves(all_moves, wait=0.08)

# Keep the local state in sync so we can derive the final front
```

path.

```python
final_state = state
for move in all_moves:
final_state = model.apply(final_state, move)
final_front = model.face_grid(final_state, "F")
final_path = find_activation_path(final_front, target)
print("front path", final_path)

client.io.sendline(b"q")
recv_for(client.io, 0.05)
client.io.sendline(b"2")
recv_for(client.io, 0.2)

transcript =
client.send_activate_steps(activation_steps(final_path), key_delay=key_delay,
read_delay=0.12)
text = transcript[-4000:]
print(decode_command_output(text) if decode_output else text)
return "hums briefly" not in transcript.lower()

print("no candidate variant found within depth", max_depth)
return False
finally:
client.close()

def run_candidate(
candidate_name: str,
max_depth: int,
key_delay: float,
host: str,
port: int,
beam_width: int,
) -> bool:
return run_target_variants(candidate_name,
CANDIDATE_RUNES[candidate_name], max_depth, key_delay, host, port, beam_width)

def run_command(
command_text: str,
max_depth: int,

key_delay: float,
host: str,
port: int,
beam_width: int,
attempts: int,
send_ascii_lines: Sequence[str],
) -> bool:
encoded = encode_command_text(command_text)
if encoded is None:
raise SystemExit("command contains unsupported characters for the
current rune mapping")
return run_rune_command(encoded, command_text, max_depth, key_delay, host,
port, beam_width, attempts, send_ascii_lines)

def run_rune_command(
rune_text: str,
display_name: str,
max_depth: int,
key_delay: float,
host: str,
port: int,
beam_width: int,
attempts: int,
send_ascii_lines: Sequence[str],
) -> bool:
encoded = rune_text

for attempt in range(1, attempts + 1):
print(f"\n=== Attempt {attempt}/{attempts}: {display_name!r} ===")
client = None
try:
client, faces = connect_and_extract(host, port)
state, encode, decode = build_state_and_lookup(faces, [encoded])
target = bytes(encode[ch] for ch in encoded)
model = FlatCubeModel()
solution = solve_target(model, state, target, max_depth,
beam_width)
print("solution", solution)
if solution is None:
continue

move_path, face_name, _ = solution
all_moves = move_path + FACE_TO_FRONT[face_name]
client.send_twist_moves(all_moves, wait=0.05)
for move in all_moves:
state = model.apply(state, move)

final_path = find_activation_path(model.face_grid(state, "F"),
target)

client.io.sendline(b"q")
recv_for(client.io, 0.05)
client.io.sendline(b"2")
recv_for(client.io, 0.2)

transcript =
client.send_activate_steps(activation_steps(final_path), key_delay=key_delay,
read_delay=0.08)
print(decode_command_output(transcript))
if "hums briefly" in transcript.lower():
continue

for line in send_ascii_lines:
client.io.sendline(line.encode())
time.sleep(0.2)
follow = recv_for(client.io, 1.0).decode("utf-8", "replace")
print(f"\n[ascii] {line}")
print(decode_command_output(follow))
return True
except Exception as exc:
print(f"[attempt {attempt}] error: {type(exc).__name__}: {exc}")
finally:
if client is not None:
client.close()
return False

def dump_charset(host: str, port: int) -> None:
client, faces = connect_and_extract(host, port)
try:
print("Observed rune charset:")
for line in describe_charset(faces):
print(" ", line)
finally:
client.close()

def main() -> int:
if hasattr(sys.stdout, "reconfigure"):
sys.stdout.reconfigure(encoding="utf-8", errors="backslashreplace")
if hasattr(sys.stderr, "reconfigure"):
sys.stderr.reconfigure(encoding="utf-8", errors="backslashreplace")

default_host = HOST
default_port = PORT

parser = argparse.ArgumentParser(description="Search and test candidate
spells for the artifact service.")
parser.add_argument("--host", default=default_host, help="Challenge host.")
parser.add_argument("--port", type=int, default=default_port,
help="Challenge port.")
parser.add_argument(
"--word",
default="time",
choices=sorted(CANDIDATE_RUNES),
help="Candidate spell family to test.",
)
parser.add_argument(
"--words",
default="",
help="Comma-separated candidate names to test in sequence. Overrides --
word when set.",
)
parser.add_argument(
"--command",
default="",
help="Plaintext command to encode with the story-cipher mapping and
execute.",
)
parser.add_argument(
"--rune-command",
default="",
help="Exact rune string to execute without plaintext encoding.",
)
parser.add_argument(
"--attempts",
type=int,
default=1,
help="Reconnect this many times when using --command.",
)
parser.add_argument(
"--send-ascii",
default="",
help="ASCII lines to send after a successful --command, separated by
'|||'.",
)
parser.add_argument("--depth", type=int, default=4, help="Maximum twist
depth to search.")
parser.add_argument("--beam-width", type=int, default=220, help="Beam
width for long targets.")
parser.add_argument("--key-delay", type=float, default=0.12, help="Delay
between activate-mode key presses.")
parser.add_argument(

"--dump-charset",
action="store_true",
help="Connect once and print observed runes with command/plaintext
hints.",
)
args = parser.parse_args()

if args.dump_charset:
dump_charset(args.host, args.port)
return 0

if args.command or args.rune_command:
send_ascii_lines = [part for part in args.send_ascii.split("|||") if
part] if args.send_ascii else []
if args.command:
run_command(
args.command,
args.depth,
args.key_delay,
args.host,
args.port,
args.beam_width,
args.attempts,
send_ascii_lines,
)
else:
run_rune_command(
args.rune_command,
args.rune_command,
args.depth,
args.key_delay,
args.host,
args.port,
args.beam_width,
args.attempts,
send_ascii_lines,
)
return 0

if args.words.strip():
for name in [part.strip() for part in args.words.split(",") if
part.strip()]:
if name not in CANDIDATE_RUNES:
raise SystemExit(f"unknown candidate: {name}")
print(f"\n=== Testing {name} ===")
if run_candidate(name, args.depth, args.key_delay, args.host,
args.port, args.beam_width):

print(f"\nCandidate {name} did not produce the failure
message.")
return 0
return 0

run_candidate(args.word, args.depth, args.key_delay, args.host, args.port,
args.beam_width)
return 0

if __name__ == "__main__":
raise SystemExit(main())
```

## 方法总结
- 核心技巧：替换文字映射 + 交互状态搜索
- 识别信号：附件是看似随机符文，但能对应自然语言；交互端会根据状态选择/执行字符。
- 复用要点：先恢复符文到明文的映射，再把 artifact 操作建模为状态搜索，生成读取上级目录 flag 的命令。
