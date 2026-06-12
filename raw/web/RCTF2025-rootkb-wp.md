# RootKB

## 题目简述

题目环境中的工具沙箱会通过 `LD_PRELOAD` 加载 `sandbox.so`，且沙箱目录可写。解法核心是覆盖该共享库，让沙箱运行时自动执行构造函数，把 `/root/flag` 移到 Web 可访问位置。

## 解题过程

### 关键观察

题目环境中的工具沙箱会通过 `LD_PRELOAD` 加载 `sandbox.so`，且沙箱目录可写。

### 求解步骤

ok了本地出了,sandbox下可写，覆盖sandbox.so，靠它运行沙盒环境时添加的ld_preload触发就
行了
exp.c
在tools那里写python代码执行写入即可
RCTF{trust_no_tool___dont_be_a_root_fool}

### PDF 外链

- <http://xn--sandbox-m31rp95k.so/>

## 方法总结

- 核心技巧：覆盖可写沙箱共享库并利用 `LD_PRELOAD` 构造函数执行。
- 识别信号：沙箱目录可写、运行时自动加载固定 `.so`。
- 复用要点：先验证文件写入和加载路径，再把读 flag 动作放进 constructor。
