# SIMD

## 题目简述

题目用 AVX2 SIMD 指令实现并行 SM4 校验。逆向难点是理解 `vmovdqu`、`vpcmpeqb`、`vpgatherdd`、`vpshufb` 等指令如何把 flag 分散到 256-bit 寄存器中并并行执行分组密码。通过常量和轮密钥定位可识别出 SM4，提取轮密钥和程序中按寄存器交错排列的密文，按真实分布重排后用 SM4 解密即可得到 flag。

题目资料地址：https://github.com/fa1conn/D3CTF-2019-Rev-SIMD-Source-Code

`source/main.c` 中 SM4 key 固定为 `01 23 45 67 89 ab cd ef fe dc ba 98 76 54 32 10`，`INITIALIZER(initialize)` 在 `main` 前执行 `change_ck()`、`sm4_setkey_enc(&ctx, key)` 并注册 `atexit(finalize)`。读取 flag 时使用 `_mm256_i32gather_epi32` 和 `vindex = {0,4,8,12,0,4,8,12}` 分组取数，再用 `_mm256_shuffle_epi8` 做字节序调整，最后并行调用 `sm4_encrypt_ecb(ctx.sk, &R0, &R1, &R2, &R3)`。这些源码细节直接对应正文里的 AVX2 数据分布和 SM4 识别。

## 解题过程

avx2网上找有很多资料，放点比较重要的

```bash
vmovdqu ymm2/m256, ymm1:Move unaligned packed integer values from ymm1 to ymm2/m256.    --->_mm256_setr_epi32 (or similar)
VPCMPEQB ymm1, ymm2, ymm3 /m256:Compare packed bytes in ymm3/m256 and ymm2 for equality.  
vpgatherdd       --->    _mm256_i32gather_epi32
```
```
//vpgatherdd example
MASK[MAXVL-1:256] ← 0;
FOR j←0 to 7
    i←j * 32;
    IF MASK[31+i] THEN
        MASK[i +31:i]←FFFFFFFFH; // extend from most significant bit
    ELSE
        MASK[i +31:i]←0;
    FI;
ENDFOR
FOR j←0 to 7
    i←j * 32;
    DATA_ADDR←BASE_ADDR + (SignExtend(VINDEX1[i+31:i])*SCALE + DISP;
    IF MASK[31+i] THEN
        DEST[i +31:i]←FETCH_32BITS(DATA_ADDR); // a fault exits the instruction
    FI;
    MASK[i +31:i]←0;
ENDFOR
DEST[MAXVL-1:256] ← 0;
```

`VPSHUFB ymm1, ymm2, ymm3/m256:Shuffle bytes in ymm2 according to contents of ymm3/m256.  --->_mm256_shuffle_epi8`

flag分布:

```lua
256register1 :   flag[0:3]    flag[16:19]   flag[32:35]    flag[48:51]    repeat   repeat   repeat    repeat
256register2 :   flag[4:7]    flag[20:23]   flag[36:39]    flag[52:55]    repeat   repeat   repeat    repeat
256register3 :   flag[8:11]   flag[24:27]   flag[40:43]    flag[56:59]    repeat   repeat   repeat    repeat
256register4 :   flag[12:15]  flag[28:31]   flag[44:47]    flag[60:63]    repeat   repeat   repeat    repeat
```

SM4 是 128-bit 分组、128-bit 主密钥的 32 轮分组密码，轮函数由 S 盒非线性变换和循环移位/异或线性变换组成。公开的 SM4 快速软件实现会用 AVX2/SIMD 同时处理多组 block，常见指令包括 `vpxor`、`vpshufb`、`vpgatherdd` 等；其中 `vpgatherdd` 适合按索引从表中取 32-bit 字，`vpshufb` 适合做字节级重排。由这些 avx2 指令想到分组密码，因为这种并行指令近年来一直应用于密码加速中。逆向工程中运用的密码一般是公开的加密算法，所以会有密钥和一些加密常量，利用这点去搜索常量识别对应的加密算法,在程序的主函数中，并没有发现密钥，(因为我指定了这部分在main函数前执行)，可以先搜寻指令，静态看一下程序逻辑，或者在x32dbg中可以看到ymm寄存器，动态调试分析指令做了些什么事，可以发现vpgatherdd是本次挑战中加载数据的主要方式，那么轮密钥也一定是用这个指令加载的， `vpgatherdd ymm2, dword ptr [edx+ymm0*4], ymm1` 应当引起我们重视，跳转到edx所在地址可以发现32个int常量，这就是我们所要找的轮密钥，在这个地址下内存断点，可以跟踪到产生轮密钥的函数 `sub_412AF0` ，密钥是 `unk_54E230`,这里用了AES的置换表来异或解得sms4的置换表(具有一点点迷惑性，没啥卵用)，搜索常量发现这是sms4算法，然后注意一下计算后密文的分布情况，提取出密文进行解密即可  
PS:AAA大哥直接看出这是sm4算法，如果是这样，只需要提取出轮密钥进行解密即可  
cipher分布：

```lua
256register4 : cipher[0:3]      cipher[16:19]       cipher[32:35]       cipher[48:51]   repeat   repeat   repeat    repeat
256register3 : cipher[4:7]      cipher[20:23]       cipher[36:39]       cipher[52:55]   repeat   repeat   repeat    repeat
256register2 : cipher[8:11]     cipher[24:27]       cipher[40:43]       cipher[56:59]   repeat   repeat   repeat    repeat
256register1 : cipher[12:15]    cipher[28:31]       cipher[44:47]       cipher[60:63]   repeat   repeat   repeat    repeat
```

程序中对比的密文分布是这样的： `cipher[0:3],cipher[16:19],cipher[32:35],cipher[48:51],cipher[4:7],cipher[20:23],cipher[36:39],cipher[52:55],cipher[8:11],cipher[24:27],cipher[40:43],cipher[56:59],cipher[12:15],cipher[28:31],cipher[44:47],cipher[60:63]`  
整理顺序解密即可

```python
# -*- coding: UTF-8 -*-
# S盒
SboxTable = 
[
    0xd6, 0x90, 0xe9, 0xfe, 0xcc, 0xe1, 0x3d, 0xb7, 0x16, 0xb6, 0x14, 0xc2, 0x28, 0xfb, 0x2c, 0x05,
    0x2b, 0x67, 0x9a, 0x76, 0x2a, 0xbe, 0x04, 0xc3, 0xaa, 0x44, 0x13, 0x26, 0x49, 0x86, 0x06, 0x99,
    0x9c, 0x42, 0x50, 0xf4, 0x91, 0xef, 0x98, 0x7a, 0x33, 0x54, 0x0b, 0x43, 0xed, 0xcf, 0xac, 0x62,
    0xe4, 0xb3, 0x1c, 0xa9, 0xc9, 0x08, 0xe8, 0x95, 0x80, 0xdf, 0x94, 0xfa, 0x75, 0x8f, 0x3f, 0xa6,
    0x47, 0x07, 0xa7, 0xfc, 0xf3, 0x73, 0x17, 0xba, 0x83, 0x59, 0x3c, 0x19, 0xe6, 0x85, 0x4f, 0xa8,
    0x68, 0x6b, 0x81, 0xb2, 0x71, 0x64, 0xda, 0x8b, 0xf8, 0xeb, 0x0f, 0x4b, 0x70, 0x56, 0x9d, 0x35,
    0x1e, 0x24, 0x0e, 0x5e, 0x63, 0x58, 0xd1, 0xa2, 0x25, 0x22, 0x7c, 0x3b, 0x01, 0x21, 0x78, 0x87,
    0xd4, 0x00, 0x46, 0x57, 0x9f, 0xd3, 0x27, 0x52, 0x4c, 0x36, 0x02, 0xe7, 0xa0, 0xc4, 0xc8, 0x9e,
    0xea, 0xbf, 0x8a, 0xd2, 0x40, 0xc7, 0x38, 0xb5, 0xa3, 0xf7, 0xf2, 0xce, 0xf9, 0x61, 0x15, 0xa1,
    0xe0, 0xae, 0x5d, 0xa4, 0x9b, 0x34, 0x1a, 0x55, 0xad, 0x93, 0x32, 0x30, 0xf5, 0x8c, 0xb1, 0xe3,
    0x1d, 0xf6, 0xe2, 0x2e, 0x82, 0x66, 0xca, 0x60, 0xc0, 0x29, 0x23, 0xab, 0x0d, 0x53, 0x4e, 0x6f,
    0xd5, 0xdb, 0x37, 0x45, 0xde, 0xfd, 0x8e, 0x2f, 0x03, 0xff, 0x6a, 0x72, 0x6d, 0x6c, 0x5b, 0x51,
    0x8d, 0x1b, 0xaf, 0x92, 0xbb, 0xdd, 0xbc, 0x7f, 0x11, 0xd9, 0x5c, 0x41, 0x1f, 0x10, 0x5a, 0xd8,
    0x0a, 0xc1, 0x31, 0x88, 0xa5, 0xcd, 0x7b, 0xbd, 0x2d, 0x74, 0xd0, 0x12, 0xb8, 0xe5, 0xb4, 0xb0,
    0x89, 0x69, 0x97, 0x4a, 0x0c, 0x96, 0x77, 0x7e, 0x65, 0xb9, 0xf1, 0x09, 0xc5, 0x6e, 0xc6, 0x84,
    0x18, 0xf0, 0x7d, 0xec, 0x3a, 0xdc, 0x4d, 0x20, 0x79, 0xee, 0x5f, 0x3e, 0xd7, 0xcb, 0x39, 0x48,
]

# 常数FK
FK = [0xa3b1bac6, 0x56aa3350, 0x677d9197, 0xb27022dc] ; ENCRYPT = 0 ;DECRYPT = 1

# 固定参数CK
CK = 
[
    0x00070e15, 0x1c232a31, 0x383f464d, 0x545b6269,
    0x70777e85, 0x8c939aa1, 0xa8afb6bd, 0xc4cbd2d9,
    0xe0e7eef5, 0xfc030a11, 0x181f262d, 0x343b4249,
    0x50575e65, 0x6c737a81, 0x888f969d, 0xa4abb2b9,
    0xc0c7ced5, 0xdce3eaf1, 0xf8ff060d, 0x141b2229,
    0x30373e45, 0x4c535a61, 0x686f767d, 0x848b9299,
    0xa0a7aeb5, 0xbcc3cad1, 0xd8dfe6ed, 0xf4fb0209,
    0x10171e25, 0x2c333a41, 0x484f565d, 0x646b7279
]

def list_4_8_to_int32(key_data): 
    return int ((key_data[0] << 24) | (key_data[1] << 16) | (key_data[2] << 8) | (key_data[3]))

def n32_to_list4_8(n): 
    return [int ((n >> 24) & 0xff), int ((n >> 16) & 0xff), int ((n >> 8) & 0xff), int ((n) & 0xff)]

def shift_left_n(x, n):
    return int (int (x << n) & 0xffffffff)

def shift_logical_left(x, n):
    return shift_left_n (x, n) | int ((x >> (32 - n)) & 0xffffffff)  #两步合在一起实现了循环左移n位

def XOR(a, b):
    return list (map (lambda x, y: x ^ y, a, b))

def sbox(idx):
    return SboxTable[idx]

def extended_key_LB(ka):      #拓展密钥算法LB
    a = n32_to_list4_8 (ka)                #a是ka的每8位组成的列表
    b = [sbox (i) for i in a]               #在s盒中每8位查找后，放入列表b，再组合成int bb
    bb = list_4_8_to_int32 (b)
    rk = bb ^ (shift_logical_left (bb, 13)) ^ (shift_logical_left (bb, 23))
    return rk

def linear_transform_L(ka):      #线性变换L
    a = n32_to_list4_8 (ka)
    b = [sbox (i) for i in a]
    bb = list_4_8_to_int32 (b)     #bb是经过s盒变换的32位数
    return bb ^ (shift_logical_left (bb, 2)) ^ (shift_logical_left (bb, 10)) ^ (shift_logical_left (bb, 18)) ^ (shift_logical_left (bb, 24)) #书上公式

def sm4_round_function(x0, x1, x2, x3, rk):   #轮函数
    return (x0 ^ linear_transform_L (x1 ^ x2 ^ x3 ^ rk))

class Sm4 (object):
    def __init__(self):
        self.sk = [0] * 32
        self.mode = ENCRYPT

    def sm4_set_key(self, key_data, mode):    #先算出拓展密钥
        self.extended_key_last (key_data, mode)

    def extended_key_last(self, key, mode):   #密钥扩展算法
        MK = [0, 0, 0, 0]
        k = [0] * 36
        MK[0] = list_4_8_to_int32 (key[0:4])
        MK[1] = list_4_8_to_int32 (key[4:8])
        MK[2] = list_4_8_to_int32 (key[8:12])
        MK[3] = list_4_8_to_int32 (key[12:16])
        k[0:4] = XOR (MK, FK)
        for i in range (32):
            k[i + 4] = k[i] ^ (extended_key_LB (k[i + 1] ^ k[i + 2] ^ k[i + 3] ^ CK[i]))
        self.sk = k[4:]   #生成的32轮子密钥放到sk中

        self.mode = mode
        if mode == DECRYPT:      #解密时rki逆序
            self.sk.reverse ()

    def sm4_one_round(self, sk, in_put):   #一轮算法 ，4个32位的字=128bit=16个字节（8*16）
        item = [list_4_8_to_int32 (in_put[0:4]), list_4_8_to_int32 (in_put[4:8]), list_4_8_to_int32 (in_put[8:12]),
                list_4_8_to_int32 (in_put[12:16])]    #4字节一个字，把每4个字节变成32位的int
        x=item

        for i in range (32):
            temp=x[3]
            x[3] = sm4_round_function (x[0], x[1], x[2], x[3], sk[i]) #x[3]成为x[4]
            x[0]=x[1]
            x[1]=x[2]
            x[2]=temp

        res=x
        # res = reduce (lambda x, y: [x[1], x[2], x[3], sm4_round_function (x[0], x[1], x[2], x[3], y)],sk, item) #32轮循环加密
        res.reverse ()
        rev = map (n32_to_list4_8, res)
        out_put = []
        [out_put.extend (_) for _ in rev]
        return out_put

    def encrypt(self, input_data):
        output_data = []
        tmp = [input_data[i:i + 16] for i in range (0, len (input_data), 16)] 
        [output_data.extend (each) for each in map (lambda x: self.sm4_one_round (self.sk, x), tmp)]
        return output_data

def encrypt(mode, key, data):
    sm4_d = Sm4 ()
    sm4_d.sm4_set_key (key, mode)
    en_data = sm4_d.encrypt (data)
    return en_data

def sm4_crypt_cbc(mode, key, iv, data):
    sm4_d = Sm4 ()
    sm4_d.sm4_set_key (key, mode)
    en_data = sm4_d.sm4_crypt_cbc (iv, data)
    return en_data

if __name__ == "__main__":
    key_data = [0x01,0x23,0x45,0x67,0x89,0xab,0xcd,0xef,0xfe,0xdc,0xba,0x98,0x76,0x54,0x32,0x10]
    sm4_d = Sm4 ()
    sm4_d.sm4_set_key (key_data, DECRYPT)
    cipher = [0x3a,0x19,0xac,0xb0,0xa6,0x4e,0xb0,0x6c,0xf8,0x20,0x0f,0x50,0xbb,0xaf,0xd2,0xf5,0x04,0x8e,0x39,0x70,0xb1,0xdd,0x44,0x26,0xa0,0x73,0x87,0x9d,0x15,0xdb,0x69,0x25,0x3a,0x27,0x16,0xb6,0x8a,0xf7,0x67,0x43,0x3b,0x82,0xac,0x26,0x34,0x8f,0x40,0x54,0xad,0x7f,0x3b,0x4d,0xbb,0x8e,0x7b,0xf6,0x18,0x8d,0x55,0x73,0xa5,0xf1,0x7e,0xca]
    print("ndecode:")
    de_data = sm4_d.encrypt (cipher)
    print(bytes(de_data))
```

## 方法总结

- 核心技巧：从 SIMD 指令的数据装载和 shuffle 方式还原分组密码的并行布局，再用算法常量识别 SM4。
- 识别信号：AVX2 大量 `vpgatherdd`、`vpshufb`、S-box/CK/FK 常量、32 轮结构同时出现时，应优先按 SM4/AES 这类公开分组密码识别。
- 复用要点：SIMD 版本的密文顺序常与线性内存顺序不同，必须按寄存器 lane 分布重排；轮密钥可能由迷惑性的表异或生成，但常量搜索和动态断点仍能定位密钥扩展函数。

