# Hackergame2020 动态链接库检查器 WP

## 题目简述

服务把用户上传的 ELF 交给 Debian 10 / glibc 2.28 的 `ldd`。通常发行版的 `ldd` 不再简单执行任意解释器，但动态装载器处理恶意程序头时仍可被构造的共享对象劫持，最终在“列依赖”阶段执行 shellcode。

## 解题过程

`ldd` 本质上以 `LD_TRACE_LOADED_OBJECTS=1` 调用动态装载器。CVE-2019-1010023 / glibc bug 22851 的样例利用恶意 ELF `PT_LOAD` 段覆盖装载器自身的可执行映射，使控制流落入攻击者附加的 shellcode；这不是给 ELF 指定普通自定义 `PT_INTERP` 那么简单，所以即使新版脚本避免直接执行任意解释器，仍可能中招。

官方 `src/solution` 的构造流程为：

```shell
gcc -fPIC -shared evil.c -T evil.script -o libevil.so
nasm -fbin shellcode.asm -o shellcode.bin
gcc make_evil.c -o make_evil
./make_evil
```

`make_evil` 给共享对象扩展两个程序头：一个 RWX `PT_LOAD` 被映射到 `0x400000` 并携带 shellcode，另一个段配合形成覆盖布局。shellcode 执行：

```asm
execve("/bin/sh", ["/bin/sh", "-c", "cat /flag"], 0)
```

把生成的 `libevil.so` 上传到检查器。服务运行 `ldd` 时，动态装载器在解析映射的过程中被覆盖，直接执行 `cat /flag`，命令输出作为所谓的“依赖检查结果”返回。

## 方法总结

`ldd` 不适合处理不可信 ELF；历史实现和动态装载器本身都可能产生代码执行面。若只需静态检查直接依赖，应解析 ELF 的 `DT_NEEDED`（例如 `readelf -d` 或专用库），并在无网络、无 flag、只读文件系统、最小权限和系统调用沙箱中处理上传文件。
