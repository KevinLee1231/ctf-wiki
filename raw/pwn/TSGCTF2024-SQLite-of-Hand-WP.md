# TSGCTF2024 SQLite of Hand

## 题目简述

程序打开 `hello.db`，用 `sqlite3_prepare_v2` 编译 `select 1;` 得到一个 `sqlite3_stmt`，然后允许用户提交少于 `0x100 * 24 = 0x1800` 字节的数据。输入先放在固定地址 `0x2000000000`，再复制到堆上，最后被直接写入语句对象偏移 136 处：

```c
char *target = malloc(N_OPs * SIZE_OP);
memcpy(target, buf, n);

void **aOp = (void **)((unsigned long long)stmt + 136);
*aOp = target;

sqlite3_step(stmt);
```

偏移 136 对应 SQLite `Vdbe` 结构的 `aOp` 字段，因此用户可以替换 SQLite 虚拟数据库引擎的 opcode 数组。目标不是构造合法 SQL，而是把 VDBE 当作可利用的寄存器虚拟机，最终执行 `system("/bin/sh")`。

## 解题过程

### 1. VDBE 中可利用的数据结构

每条 `Op` 指令为 24 字节，包含 opcode、三个整数参数、`p4type` 和一个可作指针使用的 `p4` 联合体。VDBE 的寄存器是 `Mem` 数组；一个 64 位构建中的 `Mem` 约 56 字节，重要字段是：

```c
struct Mem {
    union { i64 i; /* ... */ } u;  // +0x00，整数值
    char *z;                       // +0x08，字符串指针
    int n;                         // +0x10，字符串长度
    u16 flags;                     // +0x14
    /* db、zMalloc 等字段 */
    void (*xDel)(void *);          // 动态字符串析构器
};
```

题目使用的关键 opcode 如下：

| opcode | 作用 |
| --- | --- |
| `OP_IntCopy P1 P2` | 不做类型检查，把 `aMem[P1].u.i` 复制到 `aMem[P2].u.i` |
| `OP_AddImm P1 imm` | 给整数寄存器加立即数 |
| `OP_String len P2 ... P4` | 把 `P4` 指向的字节作为字符串放入寄存器 |
| `OP_Concat P1 P2 P3` | 计算 `P3 = P2 || P1`，并可触发可预测的堆分配 |
| `OP_Goto P2` | 把虚拟机程序计数器跳到指定 opcode 索引 |
| `OP_VCheck ... P2` | 清理指定 `Mem`；若带 `MEM_Dyn`，会调用 `xDel(z)` |

官方补丁还增加了 `OP_Pack`：

```c
case OP_Pack: {
  pIn1 = &aMem[pOp->p1];
  pOut = &aMem[pOp->p2];
  i64 *buf = malloc(sizeof(i64));
  *buf = pIn1->u.i;
  pOut->z = (char *)buf;
  pOut->n = 8;
  pOut->flags = MEM_Str | MEM_Static | MEM_Term;
  break;
}
```

它把运行时得到的地址打包成 8 字节字符串。这样便能把泄露地址拼入新的 opcode 或伪 `Mem`，弥补 `OP_IntCopy` 只能修改 `Mem.u.i`、不能直接改 `Mem.z` 的限制。

### 2. 第一阶段：泄露堆地址并生成自修改 opcode

初始 payload 同时包含会立即执行的指令和放在固定 mmap 区中的 opcode 模板。`OP_IntCopy(2, 10)` 从现有 `select 1` VDBE 的寄存器布局中取出堆指针，保存到寄存器 10。随后：

```text
OP_String  -> 取伪指令的前 16 字节
OP_Pack    -> 把泄露的 8 字节堆地址变成字符串
OP_Concat  -> 拼成完整 24 字节 Op
OP_Concat  -> 接上其余伪指令并触发大块堆分配
```

`target` 与后续 `OP_Concat` 分配在同一稳定堆序列中，因此新字符串相对 `aOp` 的 opcode 索引可预测。`OP_Goto 260` 会越过原始 `target` 末端，准确落入刚拼出的伪指令序列。这里不是直接猜 ASLR 地址，而是先取得堆地址，再把它写入第二阶段 `OP_Int64.p4`。

### 3. 第二阶段：从 SQLite 基址取得 libc 基址

第二阶段的 `OP_Int64` 通过动态填入的 `p4` 解引用堆中保存的 SQLite 代码指针，并减去本构建偏移 `0x104a80` 得到 SQLite 库基址。再加上 `1060864` 定位 `getenv@GOT`。

与上一阶段相同，先用 `OP_Pack` 把 GOT 地址拼进下一条 `OP_Int64`，用 `OP_Concat` 分配第三阶段指令串，再执行 `OP_Goto 349`。第三阶段解引用 GOT 得到 libc 中的 `getenv` 地址，并减去官方 libc 对应的 `296864`，得到 libc 基址。上述常量全部依赖题目附带的 SQLite 与 libc，换版本后必须重新计算。

### 4. 伪造动态 `Mem` 并劫持析构函数

固定 mmap 区先放置 `/bin/sh\0` 和一份伪 `Mem` 模板：

```text
z      -> "/bin/sh"
n      = -1
flags  = MEM_Dyn
xDel   -> 待填入 system
```

第三阶段用 `OP_String` 取模板字节，用 `OP_IntCopy + OP_AddImm` 计算 `libc_base + system_offset`，再通过 `OP_Pack` 和 `OP_Concat` 把 `system` 地址放入 `xDel` 字段。一次较大的拼接使伪对象落到 `aMem[2016]` 对应的可预测位置。

最后执行：

```text
OP_VCheck 0 2016
```

该 opcode 会清理目标 `Mem`。由于伪对象带有 `MEM_Dyn`，清理路径调用：

```c
mem->xDel(mem->z);
```

于是实际执行 `system("/bin/sh")`。发送生成器输出的 `pwn` 字节码，取得 shell 后读取 flag：

```text
TSGCTF{pwning_some_random_virtual_machine_is_fun}
```

## 方法总结

本题把“任意 VDBE opcode”逐步放大为代码执行：先从未做边界/类型检查的寄存器操作取得堆指针，再用新增的 `OP_Pack` 把整数地址转为可拼接字节，自修改生成能解引用 SQLite GOT 的后续 opcode，最后伪造带动态析构器的 `Mem`，借正常清理流程调用 `system`。决定性主障碍是构造内存利用原语和函数指针劫持，因此归入 Pwn，而不是仅理解字节码语义的 Reverse。根本修复是绝不让不可信数据覆盖 `Vdbe.aOp`，并在解释器中严格校验 opcode 索引、寄存器编号、类型和所有间接指针。
