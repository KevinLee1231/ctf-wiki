# GreyCTF2022 - Block

## 题目简述

题目实现了自定义 16 字节分组变换。每轮依次进行字节替换、模 $256$ 加法、位置交换和异或，共执行 30 轮。所有步骤都是可逆置换，因而只需按相反顺序实现逆运算。

## 解题过程

先从 `main.py` 逐项记录一轮的顺序。逆变换必须从最后一步开始：异或仍用相同密钥；交换操作再执行一次即可复原；加法改为模 $256$ 减法；S 盒则预先构造逆表。

```python
inv_sbox = [0] * 256
for i, value in enumerate(sbox):
    inv_sbox[value] = i

def undo_round(block, round_key):
    block = xor_layer(block, round_key)
    block = swap_layer(block)          # 交换是自身的逆
    block = [(x - d) & 0xff for x, d in zip(block, delta)]
    block = [inv_sbox[x] for x in block]
    return block

for r in range(29, -1, -1):
    state = undo_round(state, round_keys[r])
```

逐块解密并拼接后，末尾出现 `02 02`，这是合法的 PKCS#7 填充，应删除而不是当作 flag 内容。结果为：

```text
grey{I_think_I_forgot_to_put_in_my_secret..._3xPDBY9Xq5PtqjVA}
```

## 方法总结

逆向自定义分组密码时，先判断每一层是否一一映射，再严格反转“层的顺序”和“轮的顺序”。常见错误是只把加法改成减法，却仍按正向顺序执行，或忘记最后的标准填充。
