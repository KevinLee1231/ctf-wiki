# d3syscall

## 题目简述

题目通过内核模块临时替换保留 syscall，把普通程序的系统调用序列变成自定义 VM 字节码。需要从 `/proc/kallsyms` 和模块初始化逻辑确认 syscall 编号到 VM 指令的映射，再用 `strace` dump 调用参数，反汇编出 MOV/ALU/PUSH/POP/reset/checkflag 语义，最后还原 flag 解密公式。

## 解题过程

本题使用内核模块动态修改系统调用，并构造了一个简单虚拟机。程序首先从

`/proc/kallsyms` 获取系统调用表地址，并通过参数传给内核模块。内核模块会注册 Linux 保留的系统调用，分别为：

335：MOV，336：ALU，337：PUSH，338：POP，339：resetreg，340：checkflag。源码如下：

```
int init(void)
{
    sys_call_table_my = (unsigned long *)(magic);
    anything_saved[0] = (int (*)(void))(sys_call_table_my[MOV]);
    anything_saved[1] = (int (*)(void))(sys_call_table_my[ALU]);
    anything_saved[2] = (int (*)(void))(sys_call_table_my[PUSH]);
    anything_saved[3] = (int (*)(void))(sys_call_table_my[POP]);
    anything_saved[4] = (int (*)(void))(sys_call_table_my[RESET]);
    orig_cr0 = clear_cr0();
    sys_call_table_my[MOV] = (unsigned long)&mov;
    sys_call_table_my[ALU] = (unsigned long)&alu;
    sys_call_table_my[PUSH] = (unsigned long)&push;
    sys_call_table_my[POP] = (unsigned long)&pop;
    sys_call_table_my[POP+1] = (unsigned long)&reset;
    sys_call_table_my[POP+2] = (unsigned long)&check;
    setback_cr0(orig_cr0);
    return 0;
}
```

使用 `strace` 命令运行程序，可以 dump 出所有系统调用：

```
syscall_0x14f(0x1, 0, 0x333231, 0xffffffffffffff80, 0, 0x557bacc99890) =
0x333231
syscall_0x14f(0x1, 0x1, 0, 0xffffffffffffff80, 0, 0x557bacc99890) = 0
syscall_0x151(0, 0x1, 0, 0xffffffffffffff80, 0, 0x557bacc99890) = 0x1
syscall_0x14f(0, 0x2, 0, 0xffffffffffffff80, 0, 0x557bacc99890) = 0x333231
syscall_0x14f(0x1, 0x1, 0x3, 0xffffffffffffff80, 0, 0x557bacc99890) = 0x3
syscall_0x150(0x4, 0x2, 0x1, 0xffffffffffffff80, 0, 0x557bacc99890) = 0x1999188
syscall_0x14f(0x1, 0x1, 0x51e7647e, 0xffffffffffffff80, 0, 0x557bacc99890) =
0x51e7647e ...
```

稍作整理后，可以得到便于处理的数据：

```
[0x14f,0x1, 0, 0x333231, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0x1, 0x1, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x151,0, 0x1, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0, 0x2, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0x1, 0x1, 0x3, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x150,0x4, 0x2, 0x1, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0x1, 0x1, 0x51e7647e, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x150,0, 0x2, 0x1, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0, 0x3, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
```

根据逆向结果直接编写反汇编脚本：

```
bytecode=[[0x14f,0x1, 0, 0x333231, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0x1, 0x1, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x151,0, 0x1, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0, 0x2, 0, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x14f,0x1, 0x1, 0x3, 0xffffffffffffff80, 0, 0x557bacc99890],
[0x150,0x4, 0x2, 0x1, 0xffffffffffffff80, 0, 0x557bacc99890], ...
]
def mov(code):
    match code[1]:
        case 0:
            print(f"mov reg[{code[2]}],reg[{code[3]}]")
        case 1:
            print(f"mov reg[{code[2]}],{code[3]}")
def alu(code):
    match code[1]:
        case 0:
            print(f"add reg[{code[2]}],reg[{code[3]}]")
        case 1:
            print(f"sub reg[{code[2]}],reg[{code[3]}]")
        case 2:
            print(f"mul reg[{code[2]}],reg[{code[3]}]")
        case 3:
            print(f"xor reg[{code[2]}],reg[{code[3]}]")
        case 4:
            print(f"shl reg[{code[2]}],reg[{code[3]}]")
        case 5:
            print(f"shr reg[{code[2]}],reg[{code[3]}]")
def push(code):
    match code[1]:
        case 0:
            print(f"push reg[{code[2]}]")
        case 1:
            print(f"push {code[2]}")
def pop(code):
    print(f"pop reg[{code[1]}]")
for i in bytecode:
    match i[0]:
        case 335:
            mov(i)
        case 336:
            alu(i)
        case 337:
            push(i)
        case 338:
            pop(i)
        case 339:
            print("resetreg")
        case 340:
            print("checkflag")
```

getflag：

```
unsigned __int64 flag_enc[] = {
0xb0800699cb89cc89,0x4764fd523fa00b19,0x396a7e6df099d700,0xb115d56bcdeaf50a,0x25
21513c985791f4,0xb03c06af93ad0be };
int dec(unsigned __int64 & rax, unsigned __int64 & rbx)
{
    rax -= ((rbx << 6) + 0x53a35337) ^ (5 * rbx + 0x9840294d) ^ (rbx -
0x5eae4751);
    rbx -= ((rax << 3) + 0x51e7647e) ^ (rax * 3 + 0xe0b4140a) ^ (rax +
0xe6978f27);
    printf("%.8s%.8s", &rax,&rbx);
    return 0;
}
int main()
{
    printf("%d\n", sizeof(unsigned long));
    dec(flag_enc[1], flag_enc[0]);
    dec(flag_enc[3], flag_enc[2]);
    dec(flag_enc[5], flag_enc[4]);
}
```

## 方法总结

- 核心技巧：内核模块 hook syscall table、自定义 syscall VM、`strace` 抽取字节码、按 syscall 编号反汇编、TEA-like 逆运算解密。
- 识别信号：用户态程序大量调用异常 syscall 编号，且内核模块注册保留 syscall 时，应把 syscall 参数当 VM 指令流处理。
- 复用要点：先恢复编号到指令语义的映射，再写小型 disassembler；不要直接追用户态汇编，否则会漏掉真正语义在内核模块里。
