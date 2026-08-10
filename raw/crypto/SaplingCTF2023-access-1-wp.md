# access-1

## 题目简述

程序先生成 8 个随机字节 DATA，但构造密钥时只使用每个字节对 7 的余数。随后这些余数决定固定十六进制字符串 DATA_2 的八轮重排，取重排结果前 8 个十六进制字符作为 4 字节材料。程序再把它拆成两个 2 字节值，各重复 8 次扩展为 AES 密钥并连续加密两层。虽然原始随机量有 64 位，真正影响密钥的状态只有 $7^8=5\,764\,801$ 种，可以穷举。

## 解题过程

从源码完整复刻密钥生成函数，枚举 8 个位置各自的余数。每得到一个候选密钥，就用题目给出的密文解密并检查是否出现已知前缀 maple：

~~~python
from itertools import product
from Crypto.Cipher import AES

DATA_2 = "a355c8b253a6f0037c9cd331c8b4b5afb7198c944e56db549612942ceec05e8c"

def shuffle_using(xs, step):
    for i in range(0, len(xs), step):
        xs.append(xs.pop(i))
    return xs

for steps in product(range(1, 8), repeat=8):
    result = list(DATA_2)
    for step in steps:
        result = shuffle_using(result, step)

    raw = bytes.fromhex("".join(result[:8]))
    k1, k2 = raw[:2] * 8, raw[2:] * 8
    pt = AES.new(k1, AES.MODE_ECB).decrypt(
        AES.new(k2, AES.MODE_ECB).decrypt(ciphertext)
    )
    if b"maple{" in pt:
        print(pt)
        break
~~~

这里不能自行设计重排公式，必须逐行照抄源码中 pop 后 append 的次序；每轮步长是 secret % 7 + 1，即 1 至 7。解密顺序必须与加密相反，先解第二层再解第一层。安全性下降来自取模造成的状态压缩，而不是 AES 本身。正确候选解出：

~~~text
maple{N0T_3N0U9H_81T5!}
~~~

## 方法总结

评估随机密钥时要计算真正进入密码算法的熵。64 位随机输入经过逐字节模 7 压缩后只剩约 $\log_2(7^8)\approx22.46$ 位，普通枚举即可覆盖。遇到复杂的洗牌或派生流程，应先找取模、截断和重复使用等熵损失，再决定是否需要攻击底层密码。
