# GlacierCTF2023 - Glacier Military Daemon

## 题目简述

通过 PoW 和队伍 JWT 后，玩家进入只读文件系统上的 zsh。`grhealth START MAX` 以 root 身份在 80 端口启动 echo daemon；发生错误时，`glacier_resilience.h` 会递增 `argv[1]` 并执行 `execv(argv[0], argv)` 重启程序。参数只用 `strtol` 检查为非负整数，没有验证整串都被消费。

## 解题过程

先进入 `/proc`，后台启动一个正常实例占用 80 端口：

```zsh
cd /proc
grhealth 0 10 &
```

随后利用 zsh 的 `ARGV0` 为第二个进程伪造 `argv[0]`。第三个参数以数字 1 开头，因此 `strtol("1/../../flag.txt", ...)` 返回 1，能通过最大重启次数检查；同时它还是从 `/proc` 出发、最终规范化到 `/flag.txt` 的路径。

```zsh
ARGV0=/bin/cat grhealth 0 1/../../flag.txt
```

第二个 daemon 在 `bind()` 时因端口已占用进入 `handle_error()`。宏调用

```c
execv(argv[0], argv);
```

此时实际执行 `/bin/cat`，而更新后的 `argv[1]` 与原 `argv[2]` 都作为参数保留。`cat` 对 `/proc/1` 一类无关参数报错不影响它继续读取 `1/../../flag.txt`，最终输出：

```text
gctf{31230_b4ckd00r3d_pr1v4t3_m1l1t4r1_c0mp4n1_74123}
```

## 方法总结

漏洞由参数多义性和不安全重启组合而成：同一字符串既被宽松地解析成整数，又在重启后成为另一程序的路径参数；可控 `argv[0]` 又把错误恢复变成任意程序执行。数值解析应检查 `endptr` 到达字符串末尾，重启目标必须使用不可控的固定绝对路径和重新构造的参数数组。
