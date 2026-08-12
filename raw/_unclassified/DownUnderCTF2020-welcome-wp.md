# DownUnderCTF 2020 - Welcome!

## 题目简述

题目给出 SSH 地址以及公开凭据 `ductf:ductf`。登录后的自定义 shell 会持续在终端随机位置、以前景色和背景色输出 `Welcome to DUCTF!`，并以较低概率把其中一条消息替换成 flag。无需人工盯着快速刷新的彩色界面，只要把 SSH 输出交给文本过滤器即可稳定提取。

## 解题过程

服务端源码的核心循环为：

```python
flag = open('./flag.txt', 'r').read().strip()

def rainbow(screen):
    while True:
        msg = 'Welcome to DUCTF!'
        if randint(0, 250) == 7:
            msg = flag
        screen.print_at(
            msg,
            randint(0, screen.width),
            randint(0, screen.height),
            colour=randint(0, screen.colours - 1),
            bg=randint(0, screen.colours - 1),
        )
        screen.refresh()
```

每轮命中 flag 的概率是 $1/251$，而循环没有延时，因此通常很快就会出现。直接连接：

```bash
ssh ductf@target -p 1337
```

密码为：

```text
ductf
```

终端控制序列会令画面不断覆盖，不适合逐行阅读。把标准输出通过正则筛选，只保留 `DUCTF{...}` 格式即可：

```bash
ssh ductf@target -p 1337 2>/dev/null \
  | grep --line-buffered -oE 'DUCTF\{[^}]+\}' \
  | head -n 1
```

若本地 SSH 因远端命令不是普通 shell 而提示没有伪终端，可显式请求 PTY：

```bash
ssh -tt ductf@target -p 1337 2>/dev/null \
  | grep --line-buffered -oE 'DUCTF\{[^}]+\}' \
  | head -n 1
```

输出为：

```text
DUCTF{w3lc0m3_t0_DUCTF_h4v3_fun!}
```

仓库中的 `challenge/flag.txt` 与官方 WP 均保存了同一结果。彩色终端截图只会重复可完整转写的文本，没有独立视觉信息，因此不保留图片。

## 方法总结

本题的重点是把动态终端表现还原成可搜索的字节流。面对大量噪声、颜色控制序列或随机位置输出时，应优先管道化处理，再用已知 flag 格式建立正则过滤；`--line-buffered` 可减少管道缓冲造成的等待，`head -n 1` 则在首次命中后结束提取。
