# One Two Three

## 题目简述

服务用 AES 生成 CTR 密钥流来加密多行英文文本，但每加密一行都把 nonce 和计数器重置为相同初值。于是所有密文都与同一段密钥流异或，形成 many-time pad。攻击者不需要恢复 AES 密钥，只需逐列推断重复使用的密钥流。

## 解题过程

对第 $j$ 个位置，所有密文满足：

$$
c_{i,j}=m_{i,j}\oplus k_j.
$$

把不同密文的第 $j$ 个字节收集在一起，枚举 256 个候选 $k_j$，对解出的字符按可打印性、空格和英语字母频率打分：

~~~python
def score(bs):
    total = 0
    for b in bs:
        if b == 0x20:
            total += 4
        elif 0x41 <= b <= 0x5a or 0x61 <= b <= 0x7a:
            total += 3
        elif 0x20 <= b < 0x7f:
            total += 1
        else:
            total -= 8
    return total

max_len = max(map(len, ciphertexts))
keystream = bytearray()
for pos in range(max_len):
    column = [ct[pos] for ct in ciphertexts if pos < len(ct)]
    key = max(range(256), key=lambda k: score(bytes(x ^ k for x in column)))
    keystream.append(key)

for ct in ciphertexts:
    print(bytes(a ^ b for a, b in zip(ct, keystream)))
~~~

自动评分可能在样本稀少的尾部出现少量误字，可以利用上下文和 maple{...} 格式手工修正。官方结果文件中的明文为：

~~~text
The flag is maple{ctr_1s_g00d_bu7_d0n7_r3p347_n0nc35}
~~~

## 方法总结

CTR 本质上把分组密码变成流密码：$c=m\oplus\text{keystream}$。同一密钥下 nonce 绝不能重复，否则两条密文异或会直接消去密钥流。实现时应让库生成唯一 nonce，并把 nonce 与密文一起保存；不要在循环中重新创建具有相同初始计数器的 cipher。
