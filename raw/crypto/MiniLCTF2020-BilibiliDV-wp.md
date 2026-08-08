# MiniLCTF2020 - BilibiliDV

## 题目简述

题目把 Bilibili 的 AV 号映射为一个伪 “DV 号”。核心不是密码强度，而是读懂自定义 Base22 表、异或与固定偏移，再按指定位置把 5 个数位嵌回模板。官方仓库没有单独 WP，但保留了完整生成器，可从源码直接恢复算法和预期 flag。

## 解题过程

生成器中的常量为：

```python
base_table = 'd59nD71EcAt38aT24eCN06'
xor_mask = 50790
increment = 114514
template = 'DV t ACD Ne  '
dynamic_index = [12, 11, 8, 4, 2]
```

先对 AV 号做：

```text
x = (av ^ 50790) + 114514
```

再把 $x$ 写成五位 Base22。第 $i$ 位是 `base_table[(x // 22**i) % 22]`，依次放到模板下标 `12, 11, 8, 4, 2`。完整实现如下：

```python
BASE = 'd59nD71EcAt38aT24eCN06'
INDEX = [12, 11, 8, 4, 2]

def encrypt(av: int) -> str:
    x = (av ^ 50790) + 114514
    out = list('DV t ACD Ne  ')
    for i, pos in enumerate(INDEX):
        out[pos] = BASE[x // (22 ** i) % 22]
    return ''.join(out)

dv = encrypt(4123456)
print(dv)
```

输出为：

```text
DVetNACDANe80
```

题目最后要求对 DV 字符串做标准 Base64：

```python
import base64
print(base64.b64encode(b'DVetNACDANe80').decode())
```

得到 `RFZldE5BQ0RBTmU4MA==`，因此 flag 为：

```text
miniLCTF{RFZldE5BQ0RBTmU4MA==}
```

## 方法总结

自定义进制题先区分“数值变换”“进制数位”“展示模板”三层。动态位置按最低位到最高位写入，不能把 `dynamic_index` 当作从高位开始。最终 Base64 只是对已经生成的 ASCII 字符串编码，不参与 DV 算法本身。
