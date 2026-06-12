# Onion

## 题目简述

附件可与 VDI 中的 server 做 diff，恢复接近完整的符号信息。核心是自定义 VM：程序读取 50 个 64-bit 输入，加载 VM 字节码，通过自制解释器、反汇编和约束恢复输入。

## 解题过程

### 关键观察

附件可与 VDI 中的 server 做 diff，恢复接近完整的符号信息。

### 求解步骤

附件可以和VDI里的server进行diff，恢复几乎100%的符号
读取50个64位整数作为密钥，存储到0x5555555C3D00
vm字节码加载到0x5555555B5D00
参数结构：
字节码种类：

字节码
参数
作用
等价于
长度
0x01
1个int16
无条件跳转
jmp
3
0x02
1个int16
!ZFLAG 时跳转
jz
3
0x03
1个int16
ZFLAG 时跳转
jnz
3
0xFF
-
结束
exit
1
0x11
1个int16
给LOTAG赋值
mov
3
0x12
1个int16
给HITAG赋值
mov
3
0x15
1个int8
从LOTAG指向的文
件地址处加载int64
到寄存器A
load_data
2
0x16
1个int8, 1个int64
将64位立即数加载
到寄存器A
load_imm
10
0x17
2个int8
mov regA, regB
mov
3
0x18
1个int8, 1个int16
从LOTAG + imm16
指向的文件地址处
加载int64到寄存器
A
load_data
4

unsigned __int8 *opcodes;
  _QWORD reg[8];
  _WORD MEM[256];
  unsigned __int16 PC;
  unsigned __int16 HIPC;
  unsigned __int16 LOTAG;
  unsigned __int16 HITAG;
  unsigned __int16 STAKPC;
  bool ZFLAG;
  char ENDFLAG;
字节码
参数
作用
等价于
长度
0x19
1个int8
把寄存器A的值写
入LOTAG指向的文
件地址
write
2
0x1A
2个int8
从寄存器B +
LOTAG指向的文件
地址读取1字节到寄
存器A
load_data
3
0x1B
2个int8
将寄存器A的低字
节写入寄存器B +
LOTAG指向的文件
地址
write
3
0x1C
1个int8
寄存器自增，如果
结果为0 则设置
ZFLAG为true
inc
2
0x1D
1个int8
寄存器自减，如果
结果为0 则设置
ZFLAG为true
dec
2
0x1E
2个int8
shr regA, imm8 ，
如果结果为0 则设
置ZFLAG为true
shr
3
0x1F
1个int8
将寄存器低 16 位加
到LOTAG
add_tag
2
0x25
2个int8
and regA, regB，如
果结果为0 则设置
ZFLAG为true
and
3
0x26
2个int8
xor regA, regB，如
果结果为0 则设置
ZFLAG为true
xor
3
0x27
2个int8
shl regA, imm8 ，
如果结果为0 则设
置ZFLAG为true
shl
3
0x29
1个int8, 1个int64
xor regA, imm64，
如果结果为0 则设
置ZFLAG为true
xor_imm
10
0x2A
1个int8, 1个int64
and regA, imm64，
如果结果为0 则设
置ZFLAG为true
and_imm
10
从寄存器B +
字节码
参数
作用
等价于
长度
0x2B
2个int8
HITAG指向的文件
地址读取1字节到寄
存器A
load_data
3
0x2C
2个int8
将寄存器A的低字
节写入寄存器B +
HITAG指向的文件
地址
write
3
0x32
1个int8, 1个int64
cmp
regA,
imm64，如果相等
则设置ZFLAG 为
true
cmp
10
0x80
-
将下一条指令地址
保存到STAKPC
save_ret_addr
1
0x81
1个int8
将STAKPC 的值+3
后存入参数索引指
向的缓存
push_ret_addr
2
0x82
1个int8
将下一指令地址写
入HIPC-2指向的文
件地址并跳转
call
2
0x83
-
跳回返回地址，恢
复HIPC
ret
1
0x84
1个int8
将寄存器A 的值
HIPC-8指向的文件
地址
pushq
2
0x85
1个int8
从HIPC指向的文件
地址取出int64赋值
给寄存器A，恢复
HIPC
popq
2
0x90
1个int8
输出1字节
printf
2
基本结构：
外层结构体：
opcode内部前26497是文件，0xE000开始是50个8字节输入，0xFFF7处有一个倒着生长的栈
（向上生长）
在代码中存在3个函数，0x104、0x116和0x2BE，0x104是校验未通过的退出函数，0x116和
0x2BE是两个不同的加密函数
第一个check仅校验输入的第三个int64（0xE010）：
VMOPCODE ptr
REGS 8 * qword
MEM 256 * word
LOPC word
HIPC word
LOTAG word
HITAG word
STAKPC word
ZFLAG bool
ENDFLAG bool
0x0 jmp 0x100
0x3 jmp 0x5a9
0x100   svstk   0x101
0x101   jmp 0x110
0x104   stdout
0x106   stdout
0x108   stdout
0x10a   stdout
0x10c   stdout
0x10e   exit
0x110   push    mem[0x1]    0x104
0x112   svstk   0x113
0x113   jmp 0x2b8
0x2b8   push    mem[0x10]   0x116
0x2ba   svstk   0x2bb
0x2bb   jmp 0x5a4
0x2be   pushq   code[0xfff5]    reg[0]
0x2c0   pushq   code[0xffed]    reg[1]
0x2c2   pushq   code[0xffe5]    reg[2]
0x2c4   pushq   code[0xffdd]    reg[3]
0x2c6   pushq   code[0xffd5]    reg[4]
0x2c8   pushq   code[0xffcd]    reg[5]
0x2ca   pushq   code[0xffc5]    reg[6]
0x2cc   pushq   code[0xffbd]    reg[7]
0x2ce   mov reg[1]  reg[0]
0x2d1   shr reg[1]  0x20
0x2d4   and_imm reg[0]  0xffffffff
0x2de   load    code[0x7200]    reg[2]
0x2e0   pushq   code[0xffb5]    reg[0]
0x2e2   mov_imm reg[0]  0x8
0x2ec   add LOTAG   regs[0]
0x2ee   popq    reg[0]  code[0xffb5]
0x2f0   load    code[0x7208]    reg[3]
0x2f2   pushq   code[0xffb5]    reg[0]
0x2f4   mov_imm reg[0]  0x8
0x2fe   pushq   code[0xffad]    reg[5]
0x300   mov_imm reg[5]  0x10
0x30a   xor_imm reg[5]  0xffffffffffffffff
0x314   pushq   code[0xffa5]    reg[6]
0x316   pushq   code[0xff9d]    reg[7]
0x318   mov_imm reg[6]  0x1
0x322   mov reg[7]  reg[5]
0x325   and reg[7]  reg[6]
0x328   xor reg[5]  reg[6]
0x32b   cmp_imm reg[7]  0x0
0x335   jnz 0x341
0x338   shl reg[7]  0x1
0x33b   mov reg[6]  reg[7]
0x33e   jmp 0x322
0x341   popq    reg[7]  code[0xff9d]
0x343   popq    reg[6]  code[0xffa5]
0x345   pushq   code[0xffa5]    reg[6]
0x347   pushq   code[0xff9d]    reg[7]
0x349   mov reg[6]  reg[5]
0x34c   mov reg[7]  reg[0]
0x34f   and reg[7]  reg[6]
0x352   xor reg[0]  reg[6]
0x355   cmp_imm reg[7]  0x0
0x35f   jnz 0x36b
0x36b   and_imm reg[0]  0xffffffffffffffff
0x375   popq    reg[7]  code[0xff9d]
0x377   popq    reg[6]  code[0xffa5]
0x379   and_imm reg[0]  0xffffffffffffffff
0x383   popq    reg[5]  code[0xffad]
0x385   add LOTAG   regs[0]
0x387   popq    reg[0]  code[0xffb5]
0x389   mov reg[4]  reg[2]
0x38c   mov reg[5]  reg[3]
0x38f   and_imm reg[2]  0xffffffff
0x399   shr reg[4]  0x20
0x39c   and_imm reg[3]  0xffffffff
0x3a6   shr reg[5]  0x20
0x3a9   pushq   code[0xffb5]    reg[7]
0x3ab   mov reg[7]  reg[3]
0x3ae   mov reg[3]  reg[4]
0x3b1   mov reg[4]  reg[7]
0x3b4   popq    reg[7]  code[0xffb5]
0x3b6   mov_imm reg[6]  0x0
0x3c0   pushq   code[0xffb5]    reg[7]
0x3c2   mov reg[7]  reg[0]
0x3c5   pushq   code[0xffad]    reg[6]
0x3c7   pushq   code[0xffa5]    reg[7]
0x3c9   mov reg[6]  reg[7]
0x3cc   mov reg[7]  reg[7]
0x3cf   shr reg[6]  0x8
0x3d2   shl reg[7]  0x18
0x3d5   xor reg[6]  reg[7]
0x3d8   and_imm reg[6]  0xffffffff
0x3e2   popq    reg[7]  code[0xffa5]
0x3e4   mov reg[7]  reg[6]
0x3e7   popq    reg[6]  code[0xffad]
0x3e9   mov reg[0]  reg[7]
0x3ec   pushq   code[0xffad]    reg[6]
0x3ee   pushq   code[0xffa5]    reg[7]
0x3f0   mov reg[6]  reg[1]
0x3f3   mov reg[7]  reg[0]
0x3f6   and reg[7]  reg[6]
0x3f9   xor reg[0]  reg[6]
0x3fc   cmp_imm reg[7]  0x0
0x406   jnz 0x412
0x409   shl reg[7]  0x1
0x40c   mov reg[6]  reg[7]
0x40f   jmp 0x3f3
0x412   and_imm reg[0]  0xffffffffffffffff
0x41c   popq    reg[7]  code[0xffa5]
0x41e   popq    reg[6]  code[0xffad]
0x420   and_imm reg[0]  0xffffffff
0x42a   xor reg[0]  reg[2]
0x42d   and_imm reg[0]  0xffffffff
0x437   mov reg[7]  reg[1]
0x43a   pushq   code[0xffad]    reg[6]
0x43c   pushq   code[0xffa5]    reg[7]
0x43e   mov reg[6]  reg[7]
0x441   mov reg[7]  reg[7]
0x444   shl reg[6]  0x3
0x447   shr reg[7]  0x1d
0x44a   xor reg[6]  reg[7]
0x44d   and_imm reg[6]  0xffffffff
0x457   popq    reg[7]  code[0xffa5]
0x459   mov reg[7]  reg[6]
0x45c   popq    reg[6]  code[0xffad]
0x45e   mov reg[1]  reg[7]
0x461   xor reg[1]  reg[0]
0x464   and_imm reg[1]  0xffffffff
0x46e   popq    reg[7]  code[0xffb5]
0x470   cmp_imm reg[6]  0x1a
0x47a   jnz 0x54a
0x47d   pushq   code[0xffb5]    reg[0]
0x47f   pushq   code[0xffad]    reg[1]
0x481   mov reg[0]  reg[3]
0x484   mov reg[1]  reg[2]
0x487   mov reg[2]  reg[6]
0x48a   pushq   code[0xffa5]    reg[7]
0x48c   mov reg[7]  reg[0]
0x48f   pushq   code[0xff9d]    reg[6]
0x491   pushq   code[0xff95]    reg[7]
0x493   mov reg[6]  reg[7]
0x496   mov reg[7]  reg[7]
0x499   shr reg[6]  0x8
0x49c   shl reg[7]  0x18
0x49f   xor reg[6]  reg[7]
0x4a2   and_imm reg[6]  0xffffffff
0x4ac   popq    reg[7]  code[0xff95]
0x4ae   mov reg[7]  reg[6]
0x4b1   popq    reg[6]  code[0xff9d]
0x4b3   mov reg[0]  reg[7]
0x4b6   pushq   code[0xff9d]    reg[6]
0x4b8   pushq   code[0xff95]    reg[7]
0x4ba   mov reg[6]  reg[1]
0x4bd   mov reg[7]  reg[0]
0x4c0   and reg[7]  reg[6]
0x4c3   xor reg[0]  reg[6]
0x4c6   cmp_imm reg[7]  0x0
0x4d0   jnz 0x4dc
0x4d3   shl reg[7]  0x1
0x4d6   mov reg[6]  reg[7]
0x4d9   jmp 0x4bd
0x4dc   and_imm reg[0]  0xffffffffffffffff
0x4e6   popq    reg[7]  code[0xff95]
0x4e8   popq    reg[6]  code[0xff9d]
0x4ea   and_imm reg[0]  0xffffffff
0x4f4   xor reg[0]  reg[2]
0x4f7   and_imm reg[0]  0xffffffff
0x501   mov reg[7]  reg[1]
0x504   pushq   code[0xff9d]    reg[6]
0x506   pushq   code[0xff95]    reg[7]
0x508   mov reg[6]  reg[7]
0x50b   mov reg[7]  reg[7]
0x50e   shl reg[6]  0x3
0x511   shr reg[7]  0x1d
0x514   xor reg[6]  reg[7]
0x517   and_imm reg[6]  0xffffffff
0x521   popq    reg[7]  code[0xff95]
0x523   mov reg[7]  reg[6]
0x526   popq    reg[6]  code[0xff9d]
0x528   mov reg[1]  reg[7]
0x52b   xor reg[1]  reg[0]
0x52e   and_imm reg[1]  0xffffffff
0x538   popq    reg[7]  code[0xffa5]
0x53a   mov reg[2]  reg[1]
0x53d   mov reg[3]  reg[4]
0x540   mov reg[4]  reg[5]
0x543   mov reg[5]  reg[0]
0x546   popq    reg[1]  code[0xffad]
0x548   popq    reg[0]  code[0xffb5]
0x54a   inc reg[6]
0x54c   cmp_imm reg[6]  0x1b
0x556   jnz 0x55c
0x559   jmp 0x3c0
0x55c   shl reg[1]  0x20
0x55f   pushq   code[0xffb5]    reg[6]
0x561   pushq   code[0xffad]    reg[7]
0x563   mov reg[6]  reg[1]
0x566   mov reg[7]  reg[0]
0x569   and reg[7]  reg[6]
0x56c   xor reg[0]  reg[6]
0x56f   cmp_imm reg[7]  0x0
0x579   jnz 0x585
0x585   and_imm reg[0]  0xffffffffffffffff
0x58f   popq    reg[7]  code[0xffad]
0x591   popq    reg[6]  code[0xffb5]
0x593   popq    reg[7]  code[0xffbd]
0x595   popq    reg[6]  code[0xffc5]
0x597   popq    reg[5]  code[0xffcd]
0x599   popq    reg[4]  code[0xffd5]
0x59b   popq    reg[3]  code[0xffdd]
0x59d   popq    reg[2]  code[0xffe5]
0x59f   popq    reg[1]  code[0xffed]
0x5a1   popq    reg[6]  code[0xfff5]
0x5a3   retn    0x705
0x5a4   push    mem[0x20]   0x2be
0x5a6   jmp 0x3
0x5a9   mov_imm LOTAG   0xe000
0x5ac   load    reg[0]  code[0xe010]
0x5b0   mov reg[5]  reg[0]
0x5b3   pushq   code[0xfff7]    reg[5]
0x5b5   mov_imm reg[5]  0x48f0e6421ac66dea
0x5bf   xor_imm reg[5]  0xffffffffffffffff
0x5c9   pushq   code[0xffef]    reg[6]
0x5cb   pushq   code[0xffe7]    reg[7]
0x5cd   mov_imm reg[6]  0x1
0x5d7   mov reg[7]  reg[5]
0x5da   and reg[7]  reg[6]
0x5dd   xor reg[5]  reg[6]
0x5e0   cmp_imm reg[7]  0x0
0x5ea   jnz 0x5f6
0x5ed   shl reg[7]  0x1
0x5f0   mov reg[6]  reg[7]
0x5f3   jmp 0x5d7
0x5f6   popq    reg[7]  code[0xffe7]
0x5f8   popq    reg[6]  code[0xffef]
0x5fa   pushq   code[0xffef]    reg[6]
0x5fc   pushq   code[0xffe7]    reg[7]
0x5fe   mov reg[6]  reg[5]
0x601   mov reg[7]  reg[0]
0x604   and reg[7]  reg[6]
0x607   xor reg[0]  reg[6]
0x60a   cmp_imm reg[7]  0x0
0x614   jnz 0x620
0x617   shl reg[7]  0x1
0x61a   mov reg[6]  reg[7]
0x61d   jmp 0x601
0x620   and_imm reg[0]  0xffffffffffffffff
0x62a   popq    reg[7]  code[0xffe7]
0x62c   popq    reg[6]  code[0xffef]
0x62e   and_imm reg[0]  0xffffffffffffffff
0x638   popq    reg[5]  code[0xfff7]
0x63a   xor_imm reg[0]  0x5074d85b9194e696
0x644   pushq   code[0xfff7]    reg[5]
0x646   mov_imm reg[5]  0x5566488c9c5cf234
0x650   xor_imm reg[5]  0xffffffffffffffff
0x65a   pushq   code[0xffef]    reg[6]
0x65c   pushq   code[0xffe7]    reg[7]
0x65e   mov_imm reg[6]  0x1
0x668   mov reg[7]  reg[5]
0x66b   and reg[7]  reg[6]
0x66e   xor reg[5]  reg[6]
0x671   cmp_imm reg[7]  0x0
0x67b   jnz 0x687
0x67e   shl reg[7]  0x1
0x681   mov reg[6]  reg[7]
call函数之前的代码等价于：
0x684   jmp 0x668
0x687   popq    reg[7]  code[0xffe7]
0x689   popq    reg[6]  code[0xffef]
0x68b   pushq   code[0xffef]    reg[6]
0x68d   pushq   code[0xffe7]    reg[7]
0x68f   mov reg[6]  reg[5]
0x692   mov reg[7]  reg[0]
0x695   and reg[7]  reg[6]
0x698   xor reg[0]  reg[6]
0x69b   cmp_imm reg[7]  0x0
0x6a5   jnz 0x6b1
0x6a8   shl reg[7]  0x1
0x6ab   mov reg[6]  reg[7]
0x6ae   jmp 0x692
0x6b1   and_imm reg[0]  0xffffffffffffffff
0x6bb   popq    reg[7]  code[0xffe7]
0x6bd   popq    reg[6]  code[0xffef]
0x6bf   and_imm reg[0]  0xffffffffffffffff
0x6c9   popq    reg[5]  code[0xfff7]
0x6cb   xor_imm reg[0]  0x8cb331163a92fc19
0x6d5   mov_imm LOTAG   0x7200
0x6d8   mov_imm reg[1]  0x36b1cc9fe433713d
0x6e2   write   code[0x7200]    reg[1]
0x6e4   pushq   code[0xfff7]    reg[0]
0x6e6   mov_imm reg[0]  0x8
0x6f0   add LOTAG   regs[0]
0x6f2   popq    reg[0]  code[0xfff7]
0x6f4   mov_imm reg[1]  0xf97646d69c84ebd8
0x6fe   write   code[0x7208]    reg[1]
0x700   mov_imm LOTAG   0x7200
0x703   call    0x2be
0x705   cmp_imm reg[0]  0xda19ba6b81c83f61
0x70f   jnz 0x715
0x712   call    0x104
R0 = inp[2]
R5 = 0x48f0e6421ac66dea
R5 = ~R5 & 0xFFFFFFFFFFFFFFFF
R6 = 1
while True:
    R7 = R5 & R6
    R5 ^= R6
    if R7 == 0: break
    R6 = R7 << 1 & 0xFFFFFFFFFFFFFFFF
进一步等价于：
函数0x2BE是一个SPECK加密，R0为明文，code[0x7200]和code[0x7208]为顺序打乱的密钥，同
构：
R6, R7 = R5, 0
while True:
    R7 = R0 & R6
    R0 ^= R6
    if R7 == 0: break
    R6 = R7 << 1 & 0xFFFFFFFFFFFFFFFF
R5, R6, R7 = inp, 0, 0
R0 ^= 0x5074d85b9194e696
R5 = 0x5566488c9c5cf234
R5 = ~R5 & 0xFFFFFFFFFFFFFFFF
R6 = 1
while True:
    R7 = R5 & R6
    R5 ^= R6
    if R7 == 0: break
    R6 = R7 << 1 & 0xFFFFFFFFFFFFFFFF
R6, R7 = R5, 0
while True:
    R7 = R0 & R6
    R0 ^= R6
    if R7 == 0: break
    R6 = R7 << 1 & 0xFFFFFFFFFFFFFFFF
R5, R6, R7 = inp, 0, 0
R0 ^= 0x8cb331163a92fc19
R0 = inp[2]
R0 -= 0x48f0e6421ac66dea
R0 ^= 0x5074d85b9194e696
R0 -= 0x5566488c9c5cf234
R0 ^= 0x8cb331163a92fc19
def enc(inp, key):
    k = [key[0] & 0xFFFFFFFF, (key[1] >> 32) & 0xFFFFFFFF, key[1] &
0xFFFFFFFF, (key[0] >> 32) & 0xFFFFFFFF]
    rk = [k[0]]
    kl = [k[3], k[2], k[1]]
    for i in range(26):
        l1 = (((((kl[0] >> 0x08) | (kl[0] << 0x18)) & 0xFFFFFFFF) + rk[i]) ^
i) & 0xFFFFFFFF
得到中间密文0x38377082e139ecb9 ，解密得到inp[2] = 0xa28f38bd0463522c ，继续执
行：
        k1 = ((((rk[i] << 0x03) | (rk[i] >> 0x1D)) & 0xFFFFFFFF) ^ l1) &
0xFFFFFFFF
        kl.pop(0)
        kl.append(l1)
        rk.append(k1)
    x = inp & 0xFFFFFFFF
    y = inp >> 32
    for i in range(27):
        x = (((((x >> 0x08) | (x << 0x18)) & 0xFFFFFFFF) + y) & 0xFFFFFFFF) ^
rk[i]
        y = (((y << 0x03) | (y >> 0x1D)) & 0xFFFFFFFF) ^ x
    return y << 32 | x

def dec(inp, key):
    k = [key[0] & 0xFFFFFFFF, (key[1] >> 32) & 0xFFFFFFFF, key[1] &
0xFFFFFFFF, (key[0] >> 32) & 0xFFFFFFFF]
    rk = [k[0]]
    kl = [k[3], k[2], k[1]]
    for i in range(26):
        l1 = (((((kl[0] >> 0x08) | (kl[0] << 0x18)) & 0xFFFFFFFF) + rk[i]) ^
i) & 0xFFFFFFFF
        k1 = ((((rk[i] << 0x03) | (rk[i] >> 0x1D)) & 0xFFFFFFFF) ^ l1) &
0xFFFFFFFF
        kl.pop(0)
        kl.append(l1)
        rk.append(k1)
    x = inp & 0xFFFFFFFF
    y = inp >> 32
    for i in range(26, -1, -1):
        y = (((y^x) >> 0x03) | ((y^x) << 0x1D)) & 0xFFFFFFFF
        x = (((((x^rk[i]) - y) & 0xFFFFFFFF) << 0x08) | ((((x^rk[i]) - y) &
0xFFFFFFFF) >> 0x18)) & 0xFFFFFFFF
    return y << 32 | x

key = [0x36b1cc9fe433713d, 0xf97646d69c84ebd8]
inp = 0xcf626ef7f00ee7fd
enc = enc(inp, key)
print(f"{hex(enc)}")
enc = 0xda19ba6b81c83f61
dec = dec(enc, key)
print(f"{hex(dec)}")
0x116是一个标准RC4，以inp[2]为密钥，存储在code[0x7100]，S盒存储在code[0x7000]，自解
密0x72A开始长度0x6057的代码块：
0x715   mov reg[0]  reg[5]
0x718   mov_imm HITAG   0x72a
0x71b   mov_imm reg[1]  0x6057
0x725   call    0x116
0x116   mov_imm LOTAG   0x7100
0x119   write   code[0x7100]    reg[0]
0x11b   mov_imm LOTAG   0x7000
0x11e   mov_imm reg[2]  0x0
0x128   write   code[LOTAG + reg[2]]    reg[2]
0x12b   inc reg[2]
0x12d   cmp_imm reg[2]  0x100
0x137   jz  0x128
0x13a   mov_imm reg[2]  0x0
0x144   mov_imm reg[3]  0x0
0x14e   mov_imm reg[7]  0xff
0x158   mov_imm LOTAG   0x7000
0x15b   load    reg[5]  code[LOTAG + reg[2]]
0x15e   mov reg[4]  reg[2]
0x161   and_imm reg[4]  0x7
0x16b   mov_imm LOTAG   0x7100
0x16e   load    reg[4]  code[LOTAG + reg[4]]
0x171   pushq   code[0xfff5]    reg[6]
0x173   pushq   code[0xffed]    reg[7]
0x175   mov reg[6]  reg[5]
0x178   mov reg[7]  reg[3]
0x17b   and reg[7]  reg[6]
0x17e   xor reg[3]  reg[6]
0x181   cmp_imm reg[7]  0x0
0x18b   jnz 0x197
0x18e   shl reg[7]  0x1
0x191   mov reg[6]  reg[7]
0x194   jmp 0x178
0x197   and_imm reg[3]  0xffffffffffffffff
0x1a1   popq    reg[7]  code[0xffed]
0x1a3   popq    reg[6]  code[0xfff5]
0x1a5   pushq   code[0xfff5]    reg[6]
0x1a7   pushq   code[0xffed]    reg[7]
0x1a9   mov reg[6]  reg[4]
0x1ac   mov reg[7]  reg[3]
0x1af   and reg[7]  reg[6]
0x1b2   xor reg[3]  reg[6]
0x1b5   cmp_imm reg[7]  0x0
0x1bf   jnz 0x1cb
0x1c2   shl reg[7]  0x1
0x1c5   mov reg[6]  reg[7]
0x1c8   jmp 0x1ac
0x1cb   and_imm reg[3]  0xffffffffffffffff
0x1d5   popq    reg[7]  code[0xffed]
0x1d7   popq    reg[6]  code[0xfff5]
0x1d9   and reg[3]  reg[7]
0x1dc   mov_imm LOTAG   0x7000
0x1df   load    reg[6]  code[LOTAG + reg[3]]
0x1e2   write   code[LOTAG + reg[2]]    reg[6]
0x1e5   write   code[LOTAG + reg[3]]    reg[5]
0x1e8   inc reg[2]
0x1ea   cmp_imm reg[2]  0x100
0x1f4   jz  0x158
0x1f7   mov_imm reg[2]  0x0
0x201   mov_imm reg[3]  0x0
0x20b   mov_imm reg[6]  0x0
0x215   cmp_imm reg[1]  0x0
0x21f   jnz 0x2b7
0x222   inc reg[2]
0x224   and reg[2]  reg[7]
0x227   mov_imm LOTAG   0x7000
0x22a   load    reg[5]  code[LOTAG + reg[2]]
0x22d   pushq   code[0xfff5]    reg[6]
0x22f   pushq   code[0xffed]    reg[7]
0x231   mov reg[6]  reg[5]
0x234   mov reg[7]  reg[3]
0x237   and reg[7]  reg[6]
0x23a   xor reg[3]  reg[6]
0x23d   cmp_imm reg[7]  0x0
0x247   jnz 0x253
0x24a   shl reg[7]  0x1
0x24d   mov reg[6]  reg[7]
0x250   jmp 0x234
0x253   and_imm reg[3]  0xffffffffffffffff
0x25d   popq    reg[7]  code[0xffed]
0x25f   popq    reg[6]  code[0xfff5]
0x261   and reg[3]  reg[7]
0x264   load    reg[0]  code[LOTAG + reg[3]]
0x267   write   code[LOTAG + reg[2]]    reg[0]
0x26a   write   code[LOTAG + reg[3]]    reg[5]
0x26d   pushq   code[0xfff5]    reg[6]
0x26f   pushq   code[0xffed]    reg[7]
0x271   mov reg[6]  reg[0]
0x274   mov reg[7]  reg[5]
后续逻辑：
0x277   and reg[7]  reg[6]
0x27a   xor reg[5]  reg[6]
0x27d   cmp_imm reg[7]  0x0
0x287   jnz 0x293
0x28a   shl reg[7]  0x1
0x28d   mov reg[6]  reg[7]
0x290   jmp 0x274
0x293   and_imm reg[5]  0xffffffffffffffff
0x29d   popq    reg[7]  code[0xffed]
0x29f   popq    reg[6]  code[0xfff5]
0x2a1   and reg[5]  reg[7]
0x2a4   load    reg[4]  code[LOTAG + reg[5]]
0x2a7   load    reg[5]  code[HITAG + reg[6]]
0x2aa   xor reg[5]  reg[4]
0x2ad   write   code[HITAG + reg[6]]    reg[5]
0x2b0   inc reg[6]
0x2b2   dec reg[1]
0x2b4   jmp 0x215
0x2b7   retn    0x727
0x727   jmp 0x72a
0x72a   mov_imm LOTAG   0xe000
0x72d   load    reg[0]  code[0xe008]
0x731   mov reg[5]  reg[0]
0x734   xor_imm reg[0]  0x95714c91bc8b306f
0x73e   xor_imm reg[0]  0x4303f92241dd9a9f
0x748   xor_imm reg[0]  0x311e18c91413b58c
0x752   pushq   code[0xfff7]    reg[5]
0x754   mov_imm reg[5]  0x8df6073d0dbbff09
0x75e   xor_imm reg[5]  0xffffffffffffffff
0x768   pushq   code[0xffef]    reg[6]
0x76a   pushq   code[0xffe7]    reg[7]
0x76c   mov_imm reg[6]  0x1
0x776   mov reg[7]  reg[5]
0x779   and reg[7]  reg[6]
0x77c   xor reg[5]  reg[6]
0x77f   cmp_imm reg[7]  0x0
0x789   jnz 0x795
0x795   popq    reg[7]  code[0xffe7]
0x797   popq    reg[6]  code[0xffef]
0x799   pushq   code[0xffef]    reg[6]
0x79b   pushq   code[0xffe7]    reg[7]
0x79d   mov reg[6]  reg[5]
0x7a0   mov reg[7]  reg[0]
0x7a3   and reg[7]  reg[6]
0x7a6   xor reg[0]  reg[6]
0x7a9   cmp_imm reg[7]  0x0
0x7b3   jnz 0x7bf
0x7b6   shl reg[7]  0x1
0x7b9   mov reg[6]  reg[7]
0x7bc   jmp 0x7a0
0x7bf   and_imm reg[0]  0xffffffffffffffff
0x7c9   popq    reg[7]  code[0xffe7]
0x7cb   popq    reg[6]  code[0xffef]
0x7cd   and_imm reg[0]  0xffffffffffffffff
0x7d7   popq    reg[5]  code[0xfff7]
0x7d9   pushq   code[0xfff7]    reg[5]
0x7db   mov_imm reg[5]  0xee5744efe81e97b7
0x7e5   xor_imm reg[5]  0xffffffffffffffff
0x7ef   pushq   code[0xffef]    reg[6]
0x7f1   pushq   code[0xffe7]    reg[7]
0x7f3   mov_imm reg[6]  0x1
0x7fd   mov reg[7]  reg[5]
0x800   and reg[7]  reg[6]
0x803   xor reg[5]  reg[6]
0x806   cmp_imm reg[7]  0x0
0x810   jnz 0x81c
0x81c   popq    reg[7]  code[0xffe7]
0x81e   popq    reg[6]  code[0xffef]
0x820   pushq   code[0xffef]    reg[6]
0x822   pushq   code[0xffe7]    reg[7]
0x824   mov reg[6]  reg[5]
0x827   mov reg[7]  reg[0]
0x82a   and reg[7]  reg[6]
0x82d   xor reg[0]  reg[6]
0x830   cmp_imm reg[7]  0x0
0x83a   jnz 0x846
0x83d   shl reg[7]  0x1
0x840   mov reg[6]  reg[7]
0x843   jmp 0x827
0x846   and_imm reg[0]  0xffffffffffffffff
0x850   popq    reg[7]  code[0xffe7]
0x852   popq    reg[6]  code[0xffef]
0x854   and_imm reg[0]  0xffffffffffffffff
0x85e   popq    reg[5]  code[0xfff7]
0x860   pushq   code[0xfff7]    reg[6]
0x862   pushq   code[0xffef]    reg[7]
0x864   mov_imm reg[6]  0xf8a82a8dbdb78c3f
0x86e   mov reg[7]  reg[0]
0x871   and reg[7]  reg[6]
0x874   xor reg[0]  reg[6]
开始对inp[1]进行检查，同构：
0x877   cmp_imm reg[7]  0x0
0x881   jnz 0x88d
0x884   shl reg[7]  0x1
0x887   mov reg[6]  reg[7]
0x88a   jmp 0x86e
0x88d   popq    reg[7]  code[0xffef]
0x88f   popq    reg[6]  code[0xfff7]
0x891   pushq   code[0xfff7]    reg[6]
0x893   pushq   code[0xffef]    reg[7]
0x895   mov_imm reg[6]  0x58e8abfc7618f5fd
0x89f   mov reg[7]  reg[0]
0x8a2   and reg[7]  reg[6]
0x8a5   xor reg[0]  reg[6]
0x8a8   cmp_imm reg[7]  0x0
0x8b2   jnz 0x8be
0x8b5   shl reg[7]  0x1
0x8b8   mov reg[6]  reg[7]
0x8bb   jmp 0x89f
0x8be   popq    reg[7]  code[0xffef]
0x8c0   popq    reg[6]  code[0xfff7]
0x8c2   xor_imm reg[0]  0x99d88c4fa4cc68aa
0x8cc   mov_imm LOTAG   0x7200
0x8cf   mov_imm reg[1]  0x8d85b3156df9f721
0x8d9   write   code[0x7200]    reg[1]
0x8db   pushq   code[0xfff7]    reg[0]
0x8dd   mov_imm reg[0]  0x8
0x8e7   add LOTAG   regs[0]
0x8e9   popq    reg[0]  code[0xfff7]
0x8eb   mov_imm reg[1]  0x28e3d33340bc0884
0x8f5   write   code[0x7208]    reg[1]
0x8f7   mov_imm LOTAG   0x7200
0x8fa   call    0x2be
0x8fc   cmp_imm reg[0]  0x659391a5dc3522b3
0x906   jnz 0x90c
0x909   call    0x104
解密得到inp[1] = 0xbf11b34d0ce941cc ，作为下一组加密代码的RC4解密密钥，之后所有的组
都符合此模式：
R0 = inp[1]
R0 ^= 0x95714c91bc8b306f
R0 ^= 0x4303f92241dd9a9f
R0 ^= 0x311e18c91413b58c
R0 -= 0x8df6073d0dbbff09
R0 -= 0xee5744efe81e97b7
R0 += 0xf8a82a8dbdb78c3f
R0 += 0x58e8abfc7618f5fd
R0 ^= 0x99d88c4fa4cc68aa
R0 &= 0xFFFFFFFFFFFFFFFF
key = [0x8d85b3156df9f721, 0x28e3d33340bc0884]
enc = speck_dec(R0, key)
0x90c   mov reg[0]  reg[5]
0x90f   mov_imm HITAG   0x921
0x912   mov_imm reg[1]  0x5e60
0x91c   call    0x116
0x91e   jmp 0x921
0x921   mov_imm LOTAG   0xe000
0x924   load    reg[0]  code[0xe0a0]
0x928   mov reg[5]  reg[0]
0x92b   pushq   code[0xfff7]    reg[6]
0x92d   pushq   code[0xffef]    reg[7]
0x92f   mov_imm reg[6]  0x52591d5fa111b92e
0x939   mov reg[7]  reg[0]
0x93c   and reg[7]  reg[6]
0x93f   xor reg[0]  reg[6]
0x942   cmp_imm reg[7]  0x0
0x94c   jnz 0x958
0x94f   shl reg[7]  0x1
0x952   mov reg[6]  reg[7]
0x955   jmp 0x939
0x958   popq    reg[7]  code[0xffef]
0x95a   popq    reg[6]  code[0xfff7]
0x95c   xor_imm reg[0]  0xc7e29a0a50ac78e3
0x966   xor_imm reg[0]  0xead13c41399fcfd6
0x970   pushq   code[0xfff7]    reg[5]
0x972   mov_imm reg[5]  0x61553c85a2f4e8b9
0x97c   xor_imm reg[5]  0xffffffffffffffff
0x986   pushq   code[0xffef]    reg[6]
0x988   pushq   code[0xffe7]    reg[7]
0x98a   mov_imm reg[6]  0x1
0x994   mov reg[7]  reg[5]
0x997   and reg[7]  reg[6]
0x99a   xor reg[5]  reg[6]
0x99d   cmp_imm reg[7]  0x0
0x9a7   jnz 0x9b3
0x9b3   popq    reg[7]  code[0xffe7]
0x9b5   popq    reg[6]  code[0xffef]
0x9b7   pushq   code[0xffef]    reg[6]
0x9b9   pushq   code[0xffe7]    reg[7]
0x9bb   mov reg[6]  reg[5]
0x9be   mov reg[7]  reg[0]
0x9c1   and reg[7]  reg[6]
0x9c4   xor reg[0]  reg[6]
0x9c7   cmp_imm reg[7]  0x0
0x9d1   jnz 0x9dd
0x9d4   shl reg[7]  0x1
0x9d7   mov reg[6]  reg[7]
0x9da   jmp 0x9be
0x9dd   and_imm reg[0]  0xffffffffffffffff
0x9e7   popq    reg[7]  code[0xffe7]
0x9e9   popq    reg[6]  code[0xffef]
0x9eb   and_imm reg[0]  0xffffffffffffffff
0x9f5   popq    reg[5]  code[0xfff7]
0x9f7   xor_imm reg[0]  0x15f909ccb556ec05
0xa01   pushq   code[0xfff7]    reg[5]
0xa03   mov_imm reg[5]  0x4674252dee87e8a2
0xa0d   xor_imm reg[5]  0xffffffffffffffff
0xa17   pushq   code[0xffef]    reg[6]
0xa19   pushq   code[0xffe7]    reg[7]
0xa1b   mov_imm reg[6]  0x1
0xa25   mov reg[7]  reg[5]
0xa28   and reg[7]  reg[6]
0xa2b   xor reg[5]  reg[6]
0xa2e   cmp_imm reg[7]  0x0
0xa38   jnz 0xa44
0xa3b   shl reg[7]  0x1
0xa3e   mov reg[6]  reg[7]
0xa41   jmp 0xa25
0xa44   popq    reg[7]  code[0xffe7]
0xa46   popq    reg[6]  code[0xffef]
0xa48   pushq   code[0xffef]    reg[6]
0xa4a   pushq   code[0xffe7]    reg[7]
0xa4c   mov reg[6]  reg[5]
0xa4f   mov reg[7]  reg[0]
0xa52   and reg[7]  reg[6]
0xa55   xor reg[0]  reg[6]
0xa58   cmp_imm reg[7]  0x0
同构下一组inp[20]：
0xa62   jnz 0xa6e
0xa65   shl reg[7]  0x1
0xa68   mov reg[6]  reg[7]
0xa6b   jmp 0xa4f
0xa6e   and_imm reg[0]  0xffffffffffffffff
0xa78   popq    reg[7]  code[0xffe7]
0xa7a   popq    reg[6]  code[0xffef]
0xa7c   and_imm reg[0]  0xffffffffffffffff
0xa86   popq    reg[5]  code[0xfff7]
0xa88   xor_imm reg[0]  0xe885b64f981d1baa
0xa92   pushq   code[0xfff7]    reg[6]
0xa94   pushq   code[0xffef]    reg[7]
0xa96   mov_imm reg[6]  0x94c6e3f48118560b
0xaa0   mov reg[7]  reg[0]
0xaa3   and reg[7]  reg[6]
0xaa6   xor reg[0]  reg[6]
0xaa9   cmp_imm reg[7]  0x0
0xab3   jnz 0xabf
0xab6   shl reg[7]  0x1
0xab9   mov reg[6]  reg[7]
0xabc   jmp 0xaa0
0xabf   popq    reg[7]  code[0xffef]
0xac1   popq    reg[6]  code[0xfff7]
0xac3   mov_imm LOTAG   0x7200
0xac6   mov_imm reg[1]  0x1d1a63b571be74bc
0xad0   write   code[0x7200]    reg[1]
0xad2   pushq   code[0xfff7]    reg[0]
0xad4   mov_imm reg[0]  0x8
0xade   add LOTAG   regs[0]
0xae0   popq    reg[0]  code[0xfff7]
0xae2   mov_imm reg[1]  0x3e36eee3aac04cfd
0xaec   write   code[0x7208]    reg[1]
0xaee   mov_imm LOTAG   0x7200
0xaf1   call    0x2be
0xaf3   cmp_imm reg[0]  0x5538224d4c7a252a
0xafd   jnz 0xb03
0xb00   call    0x104
运算
表达式举例
特征
明文
load	reg[0]	code[0xe0e8]
code[0xe
+
mov_imm	reg[6]	0x52591d5fa111b92e
mov_imm reg[6] uint64
-
mov_imm	reg[5]	0x308442bd2f0ab265
mov_imm reg[5] uint64
^
xor_imm	reg[0]	0xd701b4a09a5bde6f
xor_imm reg[0] uint64
speck
mov_imm	reg[1]	0x54a0680a97645358
mov_imm reg[1] uint64
密文
cmp_imm	reg[0]	0x66e064f4dac280d0
cmp_imm reg[0] uint64
解得inp[20] = 0xef320f9e6ae31520 ，以此类推，注意到其加密方式除了speck之外只有3种：
+、-和^，所有关键汇编分别具有特征：
因此可以提取汇编中的特征，编写脚本同构整个VM：
R0 = inp[20]
R0 = 0x3a2e507999f2831f
R0 += 0x52591d5fa111b92e
R0 ^= 0xc7e29a0a50ac78e3
R0 ^= 0xead13c41399fcfd6
R0 -= 0x61553c85a2f4e8b9
R0 ^= 0x15f909ccb556ec05
R0 -= 0x4674252dee87e8a2
R0 ^= 0xe885b64f981d1baa
R0 += 0x94c6e3f48118560b
R0 &= 0xFFFFFFFFFFFFFFFF
key = [0x1d1a63b571be74bc, 0x3e36eee3aac04cfd]
enc = speck_enc(R0, key)
import re
import time

def printhex(array):
    for i in range(len(array)):
        print(f"0x{array[i]:016x}", end="  ")
        if i % 5 == 4: print()

def b2nle(b, n): return [int.from_bytes(b[i:i+n], byteorder='little',
signed=False) for i in range(0, len(b), n)]

def n2ble(na, n):
    b = bytearray()
    for a in na: b.extend(a.to_bytes(n, byteorder='little'))
    return b

def speck_enc(inp, key):
    k = [key[0] & 0xFFFFFFFF, (key[1] >> 32) & 0xFFFFFFFF, key[1] &
0xFFFFFFFF, (key[0] >> 32) & 0xFFFFFFFF]
    rk = [k[0]]
    kl = [k[3], k[2], k[1]]
    for i in range(26):
        l1 = (((((kl[0] >> 0x08) | (kl[0] << 0x18)) & 0xFFFFFFFF) + rk[i]) ^
i) & 0xFFFFFFFF
        k1 = ((((rk[i] << 0x03) | (rk[i] >> 0x1D)) & 0xFFFFFFFF) ^ l1) &
0xFFFFFFFF
        kl.pop(0)
        kl.append(l1)
        rk.append(k1)
    x = inp & 0xFFFFFFFF
    y = inp >> 32
    for i in range(27):
        x = (((((x >> 0x08) | (x << 0x18)) & 0xFFFFFFFF) + y) & 0xFFFFFFFF) ^
rk[i]
        y = (((y << 0x03) | (y >> 0x1D)) & 0xFFFFFFFF) ^ x
    return y << 32 | x

def speck_dec(inp, key):
    k = [key[0] & 0xFFFFFFFF, (key[1] >> 32) & 0xFFFFFFFF, key[1] &
0xFFFFFFFF, (key[0] >> 32) & 0xFFFFFFFF]
    rk = [k[0]]
    kl = [k[3], k[2], k[1]]
    for i in range(26):
        l1 = (((((kl[0] >> 0x08) | (kl[0] << 0x18)) & 0xFFFFFFFF) + rk[i]) ^
i) & 0xFFFFFFFF
        k1 = ((((rk[i] << 0x03) | (rk[i] >> 0x1D)) & 0xFFFFFFFF) ^ l1) &
0xFFFFFFFF
        kl.pop(0)
        kl.append(l1)
        rk.append(k1)
    x = inp & 0xFFFFFFFF
    y = inp >> 32
    for i in range(26, -1, -1):
        y = (((y^x) >> 0x03) | ((y^x) << 0x1D)) & 0xFFFFFFFF
        x = (((((x^rk[i]) - y) & 0xFFFFFFFF) << 0x08) | ((((x^rk[i]) - y) &
0xFFFFFFFF) >> 0x18)) & 0xFFFFFFFF
    return y << 32 | x

def solve(inst):
    enc = inst[-1][1] # 密文
    key = [inst[-3][1], inst[-2][1]] # C7200和C7208
    dec = speck_dec(enc, key) # 解密speck
    for i in range(len(inst)-4,0,-1): # 解密线性运算
        if inst[i][0] == "+": dec = (dec - inst[i][1]) & 0xFFFFFFFFFFFFFFFF
        if inst[i][0] == "-": dec = (dec + inst[i][1]) & 0xFFFFFFFFFFFFFFFF
        if inst[i][0] == "^": dec = dec ^ inst[i][1]
    return dec

def parse(insts):
    for i in range(len(insts)): # 50个单句
        inst = insts[i]
        enc = inst[-1][1] # 密文
        key = [inst[-3][1], inst[-2][1]] # C7200和C7208
        idx = inst[0] # 索引
        for j in range(1,len(inst)-3):
            if inst[j][0] == "+": print(f"inps[{idx}] += {hex(inst[j][1])}")
            if inst[j][0] == "-": print(f"inps[{idx}] -= {hex(inst[j][1])}")
            if inst[j][0] == "^": print(f"inps[{idx}] ^= {hex(inst[j][1])}")
        print(f"speck_enc(inps[{idx}], key = [{hex(inst[-3][1])},
{hex(inst[-2][1])}])")
        print(f"assert inps[{idx}] == {hex(enc)}")

def extr(l):
    fl = [it for it in l if it is not None]
    idx = -1
    c = None
    for i, ln in enumerate(fl):
        if "code[0xe" in ln:
            mt = re.search(r'code\\[(0x[0-9a-fA-F]+)\\]', ln)
            if mt:
                hs = mt.group(1)
                c = (int(hs, 16) - 0xE000) // 8
                idx = i
                break
    if idx == -1: raise ValueError("Base addr not found.")
    res = [c]
    for i in range(idx + 1, len(fl)):
        ln = fl[i]
        arg = re.search(r'0x[0-9a-fA-F]{9,}', ln)
        if not arg: continue
        hv = int(arg.group(), 16)
        if "mov_imm\\treg[6]" in ln: res.append(["+", hv])
        elif "mov_imm\\treg[5]" in ln: res.append(["-", hv])
        elif "xor_imm\\treg[0]" in ln: res.append(["^", hv])
        elif "mov_imm\\treg[1]" in ln: res.append(["k", hv])
        elif "cmp_imm\\treg[0]" in ln: res.append(["c", hv])
    return res

def extr_all(l):
    fl = [it for it in l if it is not None]
    idx = []
    for i, ln in enumerate(fl):
        if "code[0xe" in ln: idx.append(i)
    if not idx: raise ValueError("No base addr found.")
    res = []
    for ci in idx:
        ln = fl[ci]
        mt = re.search(r'code\\[(0x[0-9a-fA-F]+)\\]', ln)
        if mt:
            hs = mt.group(1)
            c = (int(hs, 16) - 0xE000) // 8
            sg = [c]
            ei = len(fl)
            if idx.index(ci) + 1 < len(idx):
                ei = idx[idx.index(ci) + 1]
            for i in range(ci + 1, ei):
                ln = fl[i]
                arg = re.search(r'0x[0-9a-fA-F]{9,}', ln)
                if not arg: continue
                hv = int(arg.group(), 16)
                if "mov_imm\\treg[6]" in ln: sg.append(["+", hv])
                elif "mov_imm\\treg[5]" in ln: sg.append(["-", hv])
                elif "xor_imm\\treg[0]" in ln: sg.append(["^", hv])
                elif "mov_imm\\treg[1]" in ln: sg.append(["k", hv])
                elif "cmp_imm\\treg[0]" in ln: sg.append(["c", hv])
            res.append(sg)
    return res

def trace_vm(inp, limit):
    with open("full_vmcode", 'rb') as file: code = file.read(26497)
    code = bytearray(code) + bytearray(0xE000 - len(code))
    binp = n2ble(inp, 8)
    code += binp
    code = bytearray(code) + bytearray(0x10000 - len(code))
    PC = 0
    HIPC = 0xFFFF
    LOTAG = 0
    HITAG = 0
    STAKPC = 0
    ZFLAG = False
    ENDFLAG = 1
    regs = [0] * 8
    mem = [0] * 256
    stdout = ""
    tracelen = 0
    DEBUG, NOISY, ASM = True, False, False # 输出trace流
    FORCEJMP = False
    PASSED = [0] * len(code)
    ASMS = [None] * len(code)
    print("traceid\\taddr\\topcode\\tname\\targs and ress")
    while True:
        if PC > limit: DEBUG, ASM = True, False # 限制trace输出范围
        else: DEBUG, ASM = False, False
        PC_1 = PC
        op = code[PC]
        if DEBUG and not ASM: print(f"
{tracelen}\\t{hex(PC)}\\t{hex(op)}",end="\\t")
        if op == 0x01:
            PC = b2nle(code[PC+1:PC+3], 2)[0]
            if DEBUG: print(f"jmp\\t{hex(PC)}")
        elif op == 0x02:
            if ZFLAG == False or FORCEJMP == True:
                PC = b2nle(code[PC+1:PC+3], 2)[0]
                if DEBUG: print(f"jz\\t{hex(PC)} => JUMPED")
            else:
                if DEBUG: print(f"jz\\t{hex(b2nle(code[PC+1:PC+3], 2)[0])} =>
NOT JUMP")
                PC += 3
        elif op == 0x03:
            if ZFLAG == True or FORCEJMP == True:
                PC = b2nle(code[PC+1:PC+3], 2)[0]
                if DEBUG: print(f"jnz\\t{hex(PC)} => JUMPED")
            else:
                if DEBUG: print(f"jnz\\t{hex(b2nle(code[PC+1:PC+3], 2)[0])} =>
NOT JUMP")
                PC += 3
        elif op == 0x11:
            LOTAG = b2nle(code[PC+1:PC+3], 2)[0]
            if DEBUG: print(f"mov_imm\\tLOTAG\\t{hex(LOTAG)}")
            PC += 3
        elif op == 0x12:
            HITAG = b2nle(code[PC+1:PC+3], 2)[0]
            if DEBUG: print(f"mov_imm\\tHITAG\\t{hex(HITAG)}")
            PC += 3
        elif op == 0x15:
            regid = code[PC+1]
            regs[regid] = b2nle(code[LOTAG:LOTAG+8], 8)[0]
            if DEBUG: print(f"load\\tcode[{hex(LOTAG)}]\\treg[{regid}] =
{hex(regs[regid])}")
            PC += 2
        elif op == 0x16:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            regs[regid] = IMM
            ZFLAG = regs[REG_B] == 0
            if DEBUG and NOISY:
print(f"mov_imm\\treg[{regid}]\\t{hex(IMM)}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY:
print(f"mov_imm\\treg[{regid}]\\t{hex(IMM)}")
            PC += 10
        elif op == 0x17:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            regs[REG_A] = regs[REG_B]
            ZFLAG = regs[REG_A] == 0
            if DEBUG and NOISY: print(f"mov\\treg[{REG_A}]\\treg[{REG_B}] =
{hex(regs[REG_A])}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY: print(f"mov\\treg[{REG_A}]\\treg[{REG_B}]
= {hex(regs[REG_A])}")
            PC += 3
        elif op == 0x18:
            regid = code[PC+1]
            addr = LOTAG + b2nle(code[PC+2:PC+4], 2)[0]
            regs[regid] = b2nle(code[addr:addr+8], 8)[0]
            if DEBUG: print(f"load\\treg[{regid}]\\tcode[{hex(addr)}] =
{hex(regs[regid])}")
            PC += 4
        elif op == 0x19:
            regid = code[PC+1]
            code[LOTAG:LOTAG+8] = n2ble([regs[regid]], 8)
            if DEBUG: print(f"write\\tcode[{hex(LOTAG)}]\\treg[{regid}] =
{hex(regs[regid])}")
            PC += 2
        elif op == 0x1A:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = LOTAG + regs[REG_B]
            regs[REG_A] = code[addr]
            if DEBUG: print(f"load\\treg[{REG_A}]\\tcode[{hex(addr)}] =
{hex(regs[REG_A])}")
            PC += 3
        elif op == 0x1B:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = LOTAG + regs[REG_B]
            code[addr] = regs[REG_A] & 0xFF
            if DEBUG: print(f"write\\tcode[{hex(addr)}]\\treg[{REG_A}] =
{hex(regs[REG_A])}")
            PC += 3
        elif op == 0x1C:
            regid = code[PC+1]
            regs[regid] += 1
            ZFLAG = regs[regid] == 0
            if DEBUG and NOISY: print(f"inc\\treg[{regid}]\\tres =
{hex(regs[regid])}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY: print(f"inc\\treg[{regid}]\\tres =
{hex(regs[regid])}")
            PC += 2
        elif op == 0x1D:
            regid = code[PC+1]
            regs[regid] -= 1
            ZFLAG = regs[regid] == 0
            if DEBUG and NOISY: print(f"dec\\treg[{regid}]\\tres =
{hex(regs[regid])}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY: print(f"dec\\treg[{regid}]\\tres =
{hex(regs[regid])}")
            PC += 2
        elif op == 0x1E:
            regid = code[PC+1]
            IMM8 = code[PC+2]
            regs[regid] = regs[regid] >> IMM8
            ZFLAG = regs[regid] == 0
            if DEBUG and NOISY: print(f"shr\\treg[{regid}]\\t{hex(IMM8)}\\tres
= {hex(regs[regid])}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY:
print(f"shr\\treg[{regid}]\\t{hex(IMM8)}\\tres = {hex(regs[regid])}")
            PC += 3
        elif op == 0x1f:
            regid = code[PC+1]
            LOTAG += regs[regid] & 0xFFFF
            if DEBUG: print(f"add\\tLOTAG\\tregs[{regid}]\\tres =
{hex(LOTAG)}")
            PC += 2
        elif op == 0x25:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            regs[REG_A] &= regs[REG_B]
            ZFLAG = regs[REG_A] == 0
            if DEBUG and NOISY:
print(f"and\\treg[{REG_A}]\\treg[{REG_B}]\\tres = {hex(regs[REG_A])}\\tZFLAG =
{ZFLAG}")
            if DEBUG and not NOISY:
print(f"and\\treg[{REG_A}]\\treg[{REG_B}]\\tres = {hex(regs[REG_A])}")
            PC += 3
        elif op == 0x26:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            regs[REG_A] ^= regs[REG_B]
            ZFLAG = regs[REG_A] == 0
            if DEBUG and NOISY:
print(f"xor\\treg[{REG_A}]\\treg[{REG_B}]\\tres = {hex(regs[REG_A])}\\tZFLAG =
{ZFLAG}")
            if DEBUG and not NOISY:
print(f"xor\\treg[{REG_A}]\\treg[{REG_B}]\\tres = {hex(regs[REG_A])}")
            PC += 3
        elif op == 0x27:
            regid = code[PC+1]
            IMM8 = code[PC+2]
            regs[regid] = (regs[regid] << IMM8) & 0xFFFFFFFFFFFFFFFF
            ZFLAG = regs[regid] == 0
            if DEBUG and NOISY: print(f"shl\\treg[{regid}]\\t{hex(IMM8)}\\tres
= {hex(regs[regid])}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY:
print(f"shl\\treg[{regid}]\\t{hex(IMM8)}\\tres = {hex(regs[regid])}")
            PC += 3
        elif op == 0x29:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            regs[regid] ^= IMM
            ZFLAG = regs[regid] == 0
            if DEBUG and NOISY:
print(f"xor_imm\\treg[{regid}]\\t{hex(IMM)}\\tres = {hex(regs[regid])}\\tZFLAG
= {ZFLAG}")
            if DEBUG and not NOISY:
print(f"xor_imm\\treg[{regid}]\\t{hex(IMM)}\\tres = {hex(regs[regid])}")
            PC += 10
        elif op == 0x2A:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            regs[regid] &= IMM
            ZFLAG = regs[regid] == 0
            if DEBUG and NOISY:
print(f"and_imm\\treg[{regid}]\\t{hex(IMM)}\\tres = {hex(regs[regid])}\\tZFLAG
= {ZFLAG}")
            if DEBUG and not NOISY:
print(f"and_imm\\treg[{regid}]\\t{hex(IMM)}\\tres = {hex(regs[regid])}")
            PC += 10
        elif op == 0x2B:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = HITAG + regs[REG_B]
            regs[REG_A] = code[addr]
            if DEBUG: print(f"load\\treg[{REG_A}]\\tcode[{hex(addr)}] =
{hex(regs[REG_A])}")
            PC += 3
        elif op == 0x2C:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = HITAG + regs[REG_B]
            code[addr] = regs[REG_A] & 0xFF
            if DEBUG: print(f"write\\tcode[{hex(addr)}]\\treg[{REG_A}] =
{hex(regs[REG_A])}")
            PC += 3
        elif op == 0x32:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            ZFLAG = regs[regid] == IMM
            if DEBUG and NOISY: print(f"cmp_imm\\treg[{regid}] =
{hex(regs[regid])}\\t{hex(IMM)}\\tZFLAG = {ZFLAG}")
            if DEBUG and not NOISY: print(f"cmp_imm\\treg[{regid}] =
{hex(regs[regid])}\\t{hex(IMM)}")
            PC += 10
        elif op == 0x80:
            STAKPC = PC + 1
            if DEBUG: print(f"svstk\\t{hex(STAKPC)}")
            PC += 1
        elif op == 0x81:
            mem[code[PC+1]] = STAKPC + 3
            if DEBUG:
print(f"push\\tmem[{hex(code[PC+1])}]\\t{hex(STAKPC+3)}")
            PC += 2
        elif op == 0x82:
            addr = mem[code[PC+1]]
            RETADDR = PC + 2
            HIPC -= 2
            code[HIPC:HIPC+2] = n2ble([RETADDR], 2)
            if DEBUG: print(f"call\\tmem[{hex(code[PC+1])}] =
{hex(addr)}\\tretaddr = {hex(RETADDR)}")
            PC = addr
        elif op == 0x83:
            addr = b2nle(code[HIPC:HIPC+2], 2)[0]
            if DEBUG: print(f"retn\\t{hex(addr)}")
            HIPC += 2
            PC = addr
        elif op == 0x84:
            regid = code[PC+1]
            HIPC -= 8
            code[HIPC:HIPC+8] = n2ble([regs[regid]], 8)
            if DEBUG: print(f"pushq\\tcode[{hex(HIPC &
0xFFFF)}]\\treg[{regid}] = {hex(regs[regid])}")
            PC += 2
        elif op == 0x85:
            regid = code[PC+1]
            regs[regid] = b2nle(code[HIPC:HIPC+8], 8)[0]
            if DEBUG: print(f"popq\\treg[{regid}]\\tcode[{hex(HIPC & 0xFFFF)}]
= {hex(regs[regid])}")
            HIPC += 8
            PC += 2
        elif op == 0x90:
            addr = code[PC+1]
            stdout += chr(addr)
            if DEBUG: print("stdout")
            PC += 2
        elif op == 0xFF:
            if DEBUG: print("exit")
            break
        else:
            if DEBUG: print(f"Unknown OP {hex(op)} at {hex(PC)}")
            break
        tracelen += 1
        PASSED.append(PC_1)

    if NOISY: print(HITAG, LOTAG, HIPC, PC)
    if NOISY: print(regs)
    if NOISY: print(b2nle(code[0x7200:0x7200+8*5], 8))
    if NOISY: print(b2nle(code[0xFFF7-8*0x20:0xFFFF], 8))
    if NOISY: print(code[0x7000:0x7100].hex())
    print(tracelen)
    print(stdout)

def run_vm(inp, limit, auto = False):
    with open("full_vmcode", 'rb') as file: code = file.read(26497)
    code = bytearray(code) + bytearray(0xE000 - len(code))
    binp = n2ble(inp, 8)
    code += binp
    code = bytearray(code) + bytearray(0x10000 - len(code))
    PC = 0
    HIPC = 0xFFFF
    LOTAG = 0
    HITAG = 0
    STAKPC = 0
    ZFLAG = False
    ENDFLAG = 1
    regs = [0] * 8
    mem = [0] * 256
    stdout = ""
    tracelen = 0
    DEBUG, NOISY, ASM = False, False, True  # 输出反汇编
    FORCEJMP = False
    PASSED = [0] * len(code)
    ASMS = [None] * len(code)
    while True:
        if PC > limit: DEBUG, ASM = False, True # 限制汇编输出范围
        else: DEBUG, ASM = False, False
        PC_1 = PC
        op = code[PC]
        if op == 0x01:
            PC = b2nle(code[PC+1:PC+3], 2)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"jmp\\t{hex(PC)}"
        elif op == 0x02:
            if ZFLAG == False or FORCEJMP == True:
                PC = b2nle(code[PC+1:PC+3], 2)[0]
                if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"jz\\t{hex(PC)}"
            else:
                if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"jz\\t{hex(b2nle(code[PC+1:PC+3], 2)[0])}"
                PC += 3
        elif op == 0x03:
            if ZFLAG == True or FORCEJMP == True:
                PC = b2nle(code[PC+1:PC+3], 2)[0]
                if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"jnz\\t{hex(PC)}"
            else:
                if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"jnz\\t{hex(b2nle(code[PC+1:PC+3], 2)[0])}"
                PC += 3
        elif op == 0x11:
            LOTAG = b2nle(code[PC+1:PC+3], 2)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"mov_imm\\tLOTAG\\t{hex(LOTAG)}"
            PC += 3
        elif op == 0x12:
            HITAG = b2nle(code[PC+1:PC+3], 2)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"mov_imm\\tHITAG\\t{hex(HITAG)}"
            PC += 3
        elif op == 0x15:
            regid = code[PC+1]
            regs[regid] = b2nle(code[LOTAG:LOTAG+8], 8)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"load\\tcode[{hex(LOTAG)}]\\treg[{regid}]"
            PC += 2
        elif op == 0x16:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            regs[regid] = IMM
            ZFLAG = regs[REG_B] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"mov_imm\\treg[{regid}]\\t{hex(IMM)}"
            PC += 10
        elif op == 0x17:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            regs[REG_A] = regs[REG_B]
            ZFLAG = regs[REG_A] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"mov\\treg[{REG_A}]\\treg[{REG_B}]"
            PC += 3
        elif op == 0x18:
            regid = code[PC+1]
            addr = LOTAG + b2nle(code[PC+2:PC+4], 2)[0]
            regs[regid] = b2nle(code[addr:addr+8], 8)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"load\\treg[{regid}]\\tcode[{hex(addr)}]"
            PC += 4
        elif op == 0x19:
            regid = code[PC+1]
            code[LOTAG:LOTAG+8] = n2ble([regs[regid]], 8)
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"write\\tcode[{hex(LOTAG)}]\\treg[{regid}]"
            PC += 2
        elif op == 0x1A:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = LOTAG + regs[REG_B]
            regs[REG_A] = code[addr]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"load\\treg[{REG_A}]\\tcode[LOTAG + reg[{REG_B}]]"
            PC += 3
        elif op == 0x1B:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = LOTAG + regs[REG_B]
            code[addr] = regs[REG_A] & 0xFF
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"write\\tcode[LOTAG + reg[{REG_B}]]\\treg[{REG_A}]"
            PC += 3
        elif op == 0x1C:
            regid = code[PC+1]
            regs[regid] += 1
            ZFLAG = regs[regid] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"inc\\treg[{regid}]"
            PC += 2
        elif op == 0x1D:
            regid = code[PC+1]
            regs[regid] -= 1
            ZFLAG = regs[regid] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"dec\\treg[{regid}]"
            PC += 2
        elif op == 0x1E:
            regid = code[PC+1]
            IMM8 = code[PC+2]
            regs[regid] = regs[regid] >> IMM8
            ZFLAG = regs[regid] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"shr\\treg[{regid}]\\t{hex(IMM8)}"
            PC += 3
        elif op == 0x1f:
            regid = code[PC+1]
            LOTAG += regs[regid] & 0xFFFF
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"add\\tLOTAG\\tregs[{regid}]"
            PC += 2
        elif op == 0x25:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            regs[REG_A] &= regs[REG_B]
            ZFLAG = regs[REG_A] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"and\\treg[{REG_A}]\\treg[{REG_B}]"
            PC += 3
        elif op == 0x26:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            regs[REG_A] ^= regs[REG_B]
            ZFLAG = regs[REG_A] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"xor\\treg[{REG_A}]\\treg[{REG_B}]"
            PC += 3
        elif op == 0x27:
            regid = code[PC+1]
            IMM8 = code[PC+2]
            regs[regid] = (regs[regid] << IMM8) & 0xFFFFFFFFFFFFFFFF
            ZFLAG = regs[regid] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"shl\\treg[{regid}]\\t{hex(IMM8)}"
            PC += 3
        elif op == 0x29:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            regs[regid] ^= IMM
            ZFLAG = regs[regid] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"xor_imm\\treg[{regid}]\\t{hex(IMM)}"
            PC += 10
        elif op == 0x2A:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            regs[regid] &= IMM
            ZFLAG = regs[regid] == 0
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"and_imm\\treg[{regid}]\\t{hex(IMM)}"
            PC += 10
        elif op == 0x2B:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = HITAG + regs[REG_B]
            regs[REG_A] = code[addr]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"load\\treg[{REG_A}]\\tcode[HITAG + reg[{REG_B}]]"
            PC += 3
        elif op == 0x2C:
            REG_A = code[PC+1]
            REG_B = code[PC+2]
            addr = HITAG + regs[REG_B]
            code[addr] = regs[REG_A] & 0xFF
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"write\\tcode[HITAG + reg[{REG_B}]]\\treg[{REG_A}]"
            PC += 3
        elif op == 0x32:
            regid = code[PC+1]
            IMM = b2nle(code[PC+2:PC+10], 8)[0]
            ZFLAG = regs[regid] == IMM
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"cmp_imm\\treg[{regid}]\\t{hex(IMM)}"
            PC += 10
        elif op == 0x80:
            STAKPC = PC + 1
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"svstk\\t{hex(STAKPC)}"
            PC += 1
        elif op == 0x81:
            mem[code[PC+1]] = STAKPC + 3
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"push\\tmem[{hex(code[PC+1])}]\\t{hex(STAKPC+3)}"
            PC += 2
        elif op == 0x82:
            addr = mem[code[PC+1]]
            RETADDR = PC + 2
            HIPC -= 2
            code[HIPC:HIPC+2] = n2ble([RETADDR], 2)
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"call\\t{hex(addr)}"
            PC = addr
        elif op == 0x83:
            addr = b2nle(code[HIPC:HIPC+2], 2)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"retn\\t{hex(addr)}"
            HIPC += 2
            PC = addr
        elif op == 0x84:
            regid = code[PC+1]
            HIPC -= 8
            code[HIPC:HIPC+8] = n2ble([regs[regid]], 8)
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"pushq\\tcode[{hex(HIPC & 0xFFFF)}]\\treg[{regid}]"
            PC += 2
        elif op == 0x85:
            regid = code[PC+1]
            regs[regid] = b2nle(code[HIPC:HIPC+8], 8)[0]
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"popq\\treg[{regid}]\\tcode[{hex(HIPC & 0xFFFF)}]"
            HIPC += 8
            PC += 2
        elif op == 0x90:
            addr = code[PC+1]
            stdout += chr(addr)
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"stdout"
            PC += 2
        elif op == 0xFF:
            if ASM and PASSED[PC_1] == 0: PASSED[PC_1] = 1; ASMS[PC_1] =
f"exit"
            break
        else:
            if DEBUG: print(f"Unknown OP {hex(op)} at {hex(PC)}")
            break
        tracelen += 1
        PASSED.append(PC_1)

    if not auto:
        for i in range(len(ASMS)):
            if ASMS[i] is not None: print(f"{hex(i)}\\t{ASMS[i]}")

    return ASMS, code, HIPC, stdout

limit = 0x5A4
inps = [0xba610b6c5d80c91a, 0xbf11b34d0ce941cc, 0xa28f38bd0463522c,
0x79ed5d84199dd9cb, 0x4d9c56b2a1d77a0d,
        0xfe13c54ceb12fea8, 0x494a63fc85b9953a, 0xad1f6be84bbb4680,
0xcd05f91609d653fa, 0x55493aa141fbe86f,
        0x25bc9aff736b80a8, 0xd8817dda43824d2c, 0x5fcca9a9cb65130d,
0x6f3ed35da24dacfa, 0xb5e1534e1dc36c87,
        0xac1b4e2750778a01, 0xc8f82d07316dcd3b, 0x36646367b78c2f91,
0x9eed7637cd5eaa26, 0xff546a0085041459,
        0xef320f9e6ae31520, 0x1e00a4b9e25488f6, 0x1a9a0626a035fb9d,
0xe2f1eb0e5248cd2c, 0x8a0bf5239eed75c4,
        0x749e8082db34037d, 0xf4d25540ed584887, 0xc12422512500c887,
0x7e1a125dcfa56359, 0x497cff13eaa5bf76,
        0xd51ceddab7795459, 0xa922933b0b315a10, 0xcabd557ffa1df043,
0xe0459b855188d045, 0x82700d6f6a986873,
        0xc01552dff3a12f67, 0x0615548ece7312fb, 0x0e189fa829657913,
0x8d4c8f2124957228, 0x451572c65bcb3425,
        0x554fca602792e879, 0x4f749f6bbca2014c, 0xb1e1adc831c8d567,
0x9c73a6d3f711e66e, 0x2ab305ec4e07b0b4,
        0x98a16d274bb044d2, 0xc409de0e72c1029e, 0x5e68e47d3a360a80,
0xa1570f48caceb3dd, 0xd6ab1c9a18ebb936]

# trace_vm(inps, limit) # 输出非函数部分的trace
# run_vm(inps, 0) # 输出整个文件的反汇编

# 输出所有校验的反编译
# ASMS, code, HIPC, stdout = run_vm(inps, limit, True)
# all_inst = extr_all(ASMS)
# parse(all_inst)

# 自动化逆向
start = time.time()
inps = [0] * 50
while 0 in inps:
    ASMS, code, HIPC, stdout = run_vm(inps, limit, True)
    inst = extr(ASMS)

### 跨页补回：自动化求解收尾

limit = b2nle(code[HIPC:HIPC+2], 2)[0]
    dec = solve(inst)
    inps[inst[0]] = dec
    print(f"[{(time.time()-start):9.4f}s] inps[0x{inst[0]:02X}] =
0x{dec:016X}")
end = time.time()
print(f"Completed. Total time: {(end-start):9.4f}s.")
printhex(inps)
print("Now getting the flag...")
ASMS, code, HIPC, stdout = run_vm(inps, limit, True)
print(stdout)
RCTF{VM_ALU_SMC_RC4_SPECK!_593eb6079d2da6c187ed462b033fee34}

## 方法总结

- 核心技巧：自定义 VM 指令集恢复与自动化求解。
- 识别信号：输入被写入固定 VM 内存区，存在 PC/HIPC/LOTAG/HITAG/栈等虚拟状态。
- 复用要点：先实现解释器和反汇编，再把每个 check 自动转成约束或逆运算。
