# ciruit map

## 题目简述

题目实现了一个简化的 Yao 乱码电路。四个输入端为 1、2、3、4，中间门依次为

$$
5=1\land2,\qquad 6=3\land4,\qquad 7=5\land6,
\qquad 9=7\oplus4.
$$

每条导线分配一对 24 位标签，分别表示布尔值 0 和 1。门表中的每一行由两个输入标签加密输出标签，并额外加密常数 0 作为校验值：

```python
def garble_label(key0, key1, key2):
    gl = encrypt(key2, key0, key1)
    validation = encrypt(0, key0, key1)
    return gl, validation

def encrypt(data, key1, key2):
    data = encrypt_data(data, key1)
    return encrypt_data(data, key2)
```

最终 flag 并不直接位于电路输出中。程序把全部标签对的和依次转为字节串，取 MD5 得到 16 字节掩码，再与给定密文异或。决定性缺陷是单个标签只有 24 位，且已知明文校验值允许对双重加密使用中间相遇攻击。

## 解题过程

### 中间相遇恢复输入标签

设门表某行的校验值为

$$
v=E_{k_2}(E_{k_1}(0)).
$$

枚举全部 $2^{24}$ 个第一层密钥，保存 $E_k(0)\mapsto k$；再枚举第二层密钥并计算 $D_k(v)$。两侧中间值相同就得到候选输入标签对。这样复杂度从 $2^{48}$ 降为约 $2\times2^{24}$：

```python
def recover_pairs(validation):
    middle = {}
    for k1 in range(1 << 24):
        middle.setdefault(encrypt_data(0, k1), []).append(k1)

    result = []
    for k2 in range(1 << 24):
        value = decrypt_data(validation, k2)
        for k1 in middle.get(value, []):
            result.append((k1, k2))
    return result
```

门表行经过随机打乱，而且标签本身不携带真假位，因此候选对仍需要用门的真值表消歧。对 AND 门，只有 `(1, 1)` 产生真标签，其余三组都产生假标签；对 XOR 门，两个输入不同时产生真标签。将候选标签送入该门的四行，只有校验值为 0 的行才是有效解密，解出的 `gl` 就是相应输出标签。

从 5、6、7 三个 AND 门开始可以确定输入标签的真假语义，再根据 9 号 XOR 门补全所有输出标签。题目提示“7 为真、4 为假时 9 为真”也与 $1\oplus0=1$ 一致，可用于检查左右输入和真假顺序没有写反。

### 生成掩码并解密

恢复所有导线的 `(false_label, true_label)` 后，完全照抄生成器的序列化规则。这里不能把整数固定编码为 3 字节，因为原程序使用的是最短大端表示：

```python
from Crypto.Util.number import long_to_bytes
from hashlib import md5

chaos = b""
for wire_id in keys:                 # 保持生成器中的插入顺序
    chaos += long_to_bytes(sum(keys[wire_id]))

mask = md5(chaos).digest()
ciphertext = bytes.fromhex("1661fe85c7b01b3db1d432ad3c5ac83a")
flag_prefix = bytes(a ^ b for a, b in zip(mask, ciphertext))
print(flag_prefix)
```

这里必须指出仓库材料中的一个长度矛盾：MD5 掩码和保留的密文都只有 16 字节，题目自定义的 `xor` 又使用 `zip`，所以附件本身最多恢复 16 字节。官方 README 记录的完整 flag 是：

```text
SCTF{#@DE-is-not-EZ@#}
```

但这串文本有 22 字节；当前密文只能验证其前 16 字节 `SCTF{#@DE-is-not`。因此完整字符串属于官方题解给出的记录，不能声称由现存附件单独恢复。若复现的是原比赛服务，应以服务端实际密文长度和提交结果再次核验。

## 方法总结

这道题的突破口不是直接枚举两把 24 位密钥，而是门表公开了双重加密下的已知明文 0。对 $E_{k_2}(E_{k_1}(0))$ 建立正反向中间值表即可把 48 位联合搜索拆成两个 24 位搜索。恢复候选标签后，再用 AND/XOR 真值表确定每个标签的布尔语义，最后严格复现标签遍历顺序和整数编码方式生成 MD5 掩码。处理中间相遇题时，已知明文、密钥空间和组合方式通常比表面总密钥长度更重要；同时必须做长度一致性检查，不能把官方文档中的完整字符串误写成由较短密文完全解出。
