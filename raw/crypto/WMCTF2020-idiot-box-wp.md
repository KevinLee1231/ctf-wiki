# idiot box

## 题目简述

本题是自定义 Feistel/SPN 风格分组密码的差分分析题。静态数据给出 8 个 6 输入、4 输出的 S 盒，这些 S 盒不是双射，而且存在异常高概率差分：对形如 $00x_1x_2 00$ 的输入差分，总能找到某个 $x_1x_2$ 使输出差分为 `0000` 的概率达到 $\frac{16}{64}$。利用这一结构可构造单个 S 盒激活的两轮迭代差分特征，再做八次 1-R 差分分析恢复第六轮子密钥分段，最后逆密钥扩展解密 flag。

## 解题过程

静态数据给出了8个具有特殊结构的S盒（很蠢），对其求差分分布可发现，

在输入差分为 $00x_1x_2 00\ (x_1,x_2\in\{0,1\})$ 形式时，均存在一 $x_1x_2$，使得该输入差分对应输出差分 `0000` 的概率为 $\frac{16}{64}$。

且 S 盒六进四出，非双射关系，因此可以找到 8 条两轮迭代高概率差分特征（分别对应激活 8 个 S 盒）。

进而得到 $p=\left(\frac{16}{64}\right)^2$ 的五轮差分特征，采用八次 **1-R差分分析** 即可得到第六轮子密钥对应每个 S 盒的分段密钥，最后通过逆向密钥扩展算法进行解密。

ps：正常的六进四出S盒，不存在仅单个S盒的二轮迭代非零差分特征，因为由E扩展函数知，该迭代差分特征要求输入差分为 $00x_1x_2 00$（即明文对处于同行），所以不可能存在满足该差分输入的明文对同时满足差分输出为0

若对相邻两个 S 盒作二轮迭代差分特征，二者信噪比均在 1 左右（参数 $c$ 约取 20），但后者要求的明文对数量 $m=c/p$ 在服务器上选择明文攻击耗时会过长，因此采用前者，题目挂载阿里云测试 exp 总耗时 25min 左右。

```python
import sys
import itertools
from tqdm import tqdm
from binascii import hexlify, unhexlify
from Crypto.Util.number import bytes_to_long, long_to_bytes
from pwn import *
ip, port = sys.argv[1], sys.argv[2]
sbox = [[11, 10, 1, 3, 8, 3, 14, 13, 0, 3, 9, 2, 4, 2, 11, 4, 6, 1, 6, 13, 6, 7, 7, 0, 10, 5, 4, 5, 9, 5, 10, 10, 6, 7, 15, 4, 7, 9, 15, 12, 1, 15, 14, 11, 14, 13, 1, 13, 8, 8, 9, 2, 12, 2, 0, 12, 8, 3, 0, 15, 11, 12, 5, 14], [0, 15, 2, 8, 3, 8, 12, 12, 9, 10, 14, 13, 4, 13, 14, 14, 2, 1, 15, 1, 1, 7, 3, 1, 10, 15, 6, 4, 6, 8, 5, 15, 4, 5, 7, 11, 7, 2, 5, 9, 11, 7, 11, 14, 6, 2, 11, 3, 12, 13, 9, 3, 9, 12, 4, 8, 10, 0, 5, 0, 0, 10, 6, 13], [5, 10, 3, 12, 3, 0, 6, 15, 13, 2, 0, 15, 8, 2, 3, 13, 9, 11, 0, 6, 14, 11, 2, 10, 1, 4, 12, 1, 7, 4, 7, 15, 5, 8, 7, 12, 5, 11, 0, 12, 14, 6, 9, 8, 14, 6, 9, 3, 4, 10, 1, 2, 10, 8, 7, 13, 15, 13, 4, 1, 5, 9, 14, 11], [4, 0, 7, 7, 7, 5, 4, 1, 10, 12, 11, 11, 11, 10, 15, 3, 12, 8, 3, 0, 2, 14, 14, 13, 2, 10, 6, 4, 6, 10, 2, 3, 14, 15, 8, 15, 9, 1, 11, 7, 5, 5, 6, 13, 6, 8, 0, 1, 3, 14, 0, 2, 9, 15, 8, 12, 1, 4, 9, 13, 9, 13, 5, 12], [
    9, 10, 3, 4, 2, 10, 12, 4, 5, 12, 5, 11, 5, 9, 13, 10, 7, 11, 7, 11, 1, 3, 2, 3, 3, 7, 1, 5, 15, 13, 9, 7, 12, 8, 8, 15, 0, 6, 0, 14, 15, 8, 8, 1, 0, 1, 0, 10, 14, 2, 14, 9, 13, 11, 6, 12, 15, 13, 14, 6, 4, 6, 4, 2], [0, 8, 12, 15, 0, 8, 3, 6, 7, 15, 9, 9, 2, 15, 9, 9, 1, 12, 13, 10, 5, 10, 12, 14, 5, 7, 14, 6, 4, 7, 5, 2, 1, 6, 4, 12, 0, 1, 14, 4, 3, 13, 11, 7, 3, 6, 11, 10, 1, 14, 2, 13, 13, 8, 15, 11, 11, 0, 5, 10, 2, 4, 8, 3], [9, 15, 3, 1, 15, 1, 7, 15, 10, 4, 0, 1, 0, 0, 3, 6, 9, 10, 12, 3, 3, 1, 12, 7, 8, 5, 2, 14, 2, 9, 2, 14, 6, 12, 13, 10, 11, 13, 9, 8, 6, 8, 5, 4, 11, 8, 14, 4, 12, 7, 13, 2, 10, 7, 13, 14, 6, 5, 5, 11, 4, 0, 15, 11], [13, 3, 1, 7, 1, 12, 10, 3, 14, 12, 14, 7, 10, 15, 5, 0, 2, 4, 13, 4, 13, 0, 8, 9, 11, 9, 10, 15, 3, 9, 12, 9, 11, 2, 8, 6, 10, 14, 11, 6, 2, 0, 6, 15, 12, 15, 6, 14, 7, 4, 13, 11, 0, 4, 7, 3, 2, 5, 1, 1, 5, 8, 8, 5]]
pbox = [19, 14, 15, 3, 10, 25, 26, 20, 23, 24, 7, 2, 18, 6, 30,
        29, 1, 4, 9, 8, 27, 5, 13, 0, 21, 16, 17, 22, 12, 31, 11, 28]
dif_dist, cipher = None, None
pc_key = [2, 13, 16, 37, 34, 32, 21, 29, 15, 25, 44, 42, 18, 35, 5, 38, 39, 12, 30, 11, 7, 20,
          17, 22, 14, 10, 26, 1, 33, 46, 45, 6, 40, 41, 43, 24, 9, 47, 4, 0, 19, 28, 27, 3, 31, 36, 8, 23]
inv_pc_key = [pc_key.index(i) for i in range(48)]

def get_data():
    global cipher
    io = remote(ip, port)
    io.recvuntil('FLAG')
    io.recvline()
    cipher = io.recvline().strip()
    print(cipher)
    io.recvuntil('input')
    io.recvline()
    return io

def enc(io, pt):
    pt_hex = hex(pt)[2:].rjust(16, '0')
    io.sendline(pt_hex)
    ct_hex = io.recvline()
    ct = int(ct_hex, 16)
    return io, ct

def s(x, i):
    row = ((x & 0b100000) >> 4) + (x & 1)
    col = (x & 0b011110) >> 1
    return sbox[i][(row << 4) + col]

def p(x):
    x_bin = [int(_) for _ in bin(x)[2:].rjust(32, '0')]
    y_bin = [x_bin[pbox[i]] for i in range(32)]
    y = int(''.join([str(_) for _ in y_bin]), 2)
    return y

def e(x):
    x_bin = bin(x)[2:].rjust(32, '0')
    y_bin = ''
    idx = -1
    for i in range(8):
        for j in range(idx, idx + 6):
            y_bin += x_bin[j % 32]
        idx += 4
    return int(y_bin, 2)

def inv_e(x_in):
    x_in = bin(x_in)[2:].rjust(48, '0')
    x = ''
    for i in range(0, 48, 6):
        x += x_in[i+1:i+5]
    x = int(x, 2)
    return x

def F(x, k):
    x_in = bin(e(x) ^ k)[2:].rjust(48, '0')
    y_out = ''
    for i in range(0, 48, 6):
        x_in_i = int(x_in[i:i+6], 2)
        y_out += bin(s(x_in_i, i // 6))[2:].rjust(4, '0')
    y_out = int(y_out, 2)
    y = p(y_out)
    return y
# sub_key(bin_str) has 48-bits

def dec_block(y, sub_key):
    y_bin = bin(y)[2:].rjust(64, '0')
    l, r = int(y_bin[:32], 2), int(y_bin[32:], 2)
    for i in range(6):
        l, r = r, l ^ F(r, int(sub_key, 2))
        sub_key = ''.join([sub_key[inv_pc_key[j]] for j in range(48)])
    x = (l + (r << 32)) & ((1 << 64) - 1)
    return x

def dec(ct, sub_key):
    assert(len(ct) % 8 == 0)
    pt = b''
    for i in range(0, len(ct), 8):
        pt_block = long_to_bytes(
            dec_block(bytes_to_long(ct[i:i+8]), sub_key)).rjust(8, b'\x00')
        pt += pt_block
    return pt
# Differential distribution

def gen_dif_dist():
    global dif_dist
    dif_dist = []
    keys = list(itertools.product(range(64), repeat=2))
    for i in range(8):
        dif_dist_i = dict()
        for key in keys:
            dif_dist_i[key] = 0
        for (x, x_ast) in keys:
            x_dif = (x ^ x_ast) & 0b111111
            y_dif = (s(x, i) ^ s(x_ast, i)) & 0b1111
            dif_dist_i[(x_dif, y_dif)] += 1
        dif_dist.append(dif_dist_i)
# Find 2-round iterative features (sbox[idx])

def find_path(idx):
    max_pro = 0
    path = None
    for i in range(1, 4):
        value = dif_dist[idx][(i << 2, 0)]
        if value > max_pro:
            max_pro = value
            path = i << 2
    return path, max_pro
# Get input_dif corresponding to the path found

def input_dif(path, idx):
    x_in = '0' * idx * 6 + bin(path)[2:].rjust(6, '0') + '0' * (7 - idx) * 6
    x = inv_e(int(x_in, 2))
    return (x << 32)
# Filter wrong pairs

def filter_pair(pt_dif, idx, io):
    filt = hex(pt_dif)[2:].rjust(16, '0')[:8]
    cts = []
    i_shift = 60 - idx * 4
    j_shift = (i_shift - 4) % 32 + 32
    k_shift = (i_shift + 4) % 32 + 32
    for i in tqdm(range(2**4)):
        for j in range(2**4):
            for k in range(2**4):
                pt = (i << i_shift) + (j << j_shift) + (k << k_shift)
                pt_ast = pt ^ pt_dif
                io, ct = enc(io, pt)
                io, ct_ast = enc(io, pt_ast)
                ct_dif = ct ^ ct_ast
                if hex(ct_dif)[2:].rjust(16, '0')[8:] == filt:
                    cts.append((ct, ct_ast))
    return cts, io
# Find satisfied dif-features

def gen_features(idx, io):
    path, max_pro = find_path(idx)
    # print((path, max_pro))
    pt_dif = input_dif(path, idx)
    # print(hex(pt_dif)[2:].rjust(16, '0'))
    cts, io = filter_pair(pt_dif, idx, io)
    return cts, max_pro, io

def crack_part_key(cts, idx, m):
    sub_key = [0] * (2**6)
    if len(cts) > m:
        cts = cts[len(cts)//2-m//2:len(cts)//2+m//2]
    for (ct, ct_ast) in tqdm(cts):
        ctl = ct >> 32
        ctr = ct & ((1 << 32) - 1)
        ctl_ast = ct_ast >> 32
        ctr_ast = ct_ast & ((1 << 32) - 1)
        for i in range(2**6):
            pro_key = i << (42 - 6 * idx)
            if (F(ctr, pro_key) ^ F(ctr_ast, pro_key) ^ ctl ^ ctl_ast) == 0:
                sub_key[i] += 1
    corr_num = max(sub_key)
    pro_part_key = []
    for i in range(2**6):
        if sub_key[i] == corr_num:
            pro_part_key.append(i)
    return pro_part_key

def crack_key(io):
    pro_key = []
    for i in range(8):
        print('[+] cracking {}/8'.format(i + 1))
        cts, max_pro, io = gen_features(i, io)
        #c = 20
        #m = (c * 64 * 64) // (max_pro ** 2)
        pro_part_key = crack_part_key(cts, i, 320)
        pro_key.append(pro_part_key)
    io.close()
    return pro_key

def get_flag(pro_key):
    flag = None
    ct = unhexlify(cipher)
    sub_key = list(itertools.product(
        pro_key[0], pro_key[1], pro_key[2], pro_key[3], pro_key[4], pro_key[5], pro_key[6], pro_key[7]))
    for i in range(len(sub_key)):
        sk = 0
        for j in range(8):
            sk += (sub_key[i][j] << (42 - 6 * j))
        sub_key[i] = bin(sk)[2:].rjust(48, '0')
    for sk in tqdm(sub_key):
        pt = dec(ct, sk)
        if b'WMCTF' in pt:
            print(pt)
            flag = pt
            break
    return flag

if __name__ == '__main__':
    io = get_data()
    gen_dif_dist()
    pro_key = crack_key(io)
    print(pro_key)
    flag = get_flag(pro_key)
    print(flag)
```

## 方法总结

差分分析题的第一步是对 S 盒求差分分布表，而不是直接爆破密钥。本题的异常点在于 6 进 4 出非双射 S 盒存在单 S 盒两轮迭代高概率差分，使五轮特征概率达到可利用范围。利用时按 S 盒分段选择明文对、过滤满足差分传播的密文对、统计第六轮子密钥候选，最后把 8 个 6-bit 分段组合并逆密钥扩展验证明文中是否出现 `WMCTF`。
