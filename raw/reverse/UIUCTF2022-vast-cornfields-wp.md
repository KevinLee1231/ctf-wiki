# Vast Cornfields

## 题目简述

程序要求输入由小写字母和下划线组成的字符串，经过一套被递归函数严重拉长的变换后，与固定密文比较。成功时，程序把原始输入包进 `uiuctf{...}` 输出。

`s`、`eq`、`su`、`a`、`m`、`l`、`ev` 看起来相互递归，实际分别实现后继、相等、截断减法、加法、乘法、小于和偶数判断。把这些基础运算替换成普通表达式后，`encode()` 清楚地呈现为 Four-square cipher；下划线不参与配对，只保留在原位置。

## 解题过程

### 化简递归包装

这些函数的高层语义为：

```text
s(n)      = n + 1
eq(x, y)  = (x == y)
su(x, y)  = max(x - y, 0)
a(x, y)   = x + y
m(x, y)   = x * y
l(x, y)   = (x < y)
ev(n)     = (n % 2 == 0)
```

输入限制也随之变得直观：总长度必须为偶数，最后一个字符不能是下划线，只允许 25 字母表中的小写字母和 `_`，且非下划线字符数必须为偶数。字母 `q` 被排除，因为四方密码使用 $5\times5$ 方阵。

### 识别 Four-square 方阵

普通方阵 1 和 3 为：

```text
a b c d e
f g h i j
k l m n o
p r s t u
v w x y z
```

密钥方阵 2、4 按行展开分别是：

```text
vastbcdefghijklmnopruwxyz
cornfieldsabghjkmptuvwxyz
```

对一对明文字母 $p_1,p_2$，若它们在普通方阵中的坐标为 $(x_1,y_1)$、$(x_2,y_2)$，加密结果就是：

$$
c_1=S_2[x_1,y_2],\qquad c_2=S_4[x_2,y_1].
$$

因此解密时，在 $S_2$ 中查 $c_1$ 得到 $(x_1,y_2)$，在 $S_4$ 中查 $c_2$ 得到 $(x_2,y_1)$，再从普通方阵取回 $(x_1,y_1)$ 和 $(x_2,y_2)$。

### 解密固定目标

目标密文为：

```text
odt_sjtfnb_jc_c_fiajb_he_ciuh_nkn_atvfjp
```

保持下划线位置不动，只把其余字符按出现顺序两两解密：

```python
alphabet = "abcdefghijklmnoprstuvwxyz"
square2 = "vastbcdefghijklmnopruwxyz"
square4 = "cornfieldsabghjkmptuvwxyz"

ciphertext = "odt_sjtfnb_jc_c_fiajb_he_ciuh_nkn_atvfjp"
plaintext = ["_" for _ in ciphertext]
positions = [i for i, ch in enumerate(ciphertext) if ch != "_"]

for i in range(0, len(positions), 2):
    first, second = positions[i], positions[i + 1]

    row1, col2 = divmod(square2.index(ciphertext[first]), 5)
    row2, col1 = divmod(square4.index(ciphertext[second]), 5)

    plaintext[first] = alphabet[5 * row1 + col1]
    plaintext[second] = alphabet[5 * row2 + col2]

answer = "".join(plaintext)
print(answer)
```

输出为：

```text
the_inside_of_a_field_of_corn_and_dreams
```

该字符串长度为 40，含 32 个非下划线字符，满足程序的长度、字符集和偶数约束。把它提交给二进制后，正向加密结果与固定密文完全一致，最终得到：

```text
uiuctf{the_inside_of_a_field_of_corn_and_dreams}
```

## 方法总结

- 核心技巧：先给递归基础函数恢复语义，再识别两个普通方阵和两个密钥方阵组成的 Four-square cipher，保留下划线位置逆向解密。
- 识别信号：代码反复通过递归实现加减乘、比较和奇偶判断，核心编码却在两个 $5\times5$ 查表之间交换行列坐标；25 字母表还明确排除了 `q`。
- 复用要点：不要在递归包装中逐条模拟到最后，应先做函数摘要并替换为高层运算。处理带分隔符的古典密码时，还要区分“字符串下标”和“参与密码配对的字符序号”。
