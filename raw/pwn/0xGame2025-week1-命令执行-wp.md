# 命令执行🤔

## 题目简述

程序读取最多 `0x20` 字节并直接传给 `system()`，只在字符串层面拒绝包含连续子串 `cat` 或 `sh` 的输入。漏洞本质是命令执行接口前的脆弱黑名单，而不是对允许命令进行白名单校验。

## 解题过程

关键源码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/ioctl.h>

void init()
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IOLBF, 0);
    setvbuf(stderr, NULL, _IOFBF, 1024);
}

int main()
{
    char s[0x30];
    init();
    puts("Please input your command,no cat no sh!");
    read(0,s,0x20);
    if(strstr(s,"cat")||strstr(s,"sh"))
    {
        puts("No way!");
        exit(0);
    }
    system(s);
    return 0;
}
```

`strstr()` 检查的是原始输入，而 Shell 随后还会进行引号拼接、参数展开和命令解析。因此只要让输入文本不连续出现被禁子串、但解析结果仍是目标命令，即可绕过。

例如单引号包围空字符串或部分字符，Shell 会把相邻片段拼成同一个命令：

```sh
c''at flag
```

原始文本是 `c''at`，不含连续的 `cat`，但 Shell 去除空引号后执行 `cat flag`。也可以直接使用没有进入黑名单的等价读取命令，如 `tac flag`。

另一种方式是输入：

```sh
$0
```

`system()` 实际通过 `/bin/sh -c` 执行命令；在该 Shell 中，`$0` 展开为当前 Shell 名称，执行后得到一个新的交互 Shell。

## 方法总结

- 核心技巧：利用 Shell 解析发生在字符串黑名单之后，构造“文本不同、语义相同”的命令。
- 识别信号：用户输入直接进入 `system()`，过滤仅由少量 `strstr()` 子串检查组成。
- 复用要点：从引号拼接、变量展开、通配符和等价命令角度检查黑名单；防御端应避免调用 Shell，或使用严格的命令及参数白名单。
