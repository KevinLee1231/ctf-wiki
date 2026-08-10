# HGAME 2025 Ancient Recall

## 题目简述

题目把五张塔罗牌编码成整数向量，并把相邻两项相加的 `Fortune_wheel` 连续执行 250 次，只给出最终的五个大整数。由于每一轮都是固定的线性变换，250 轮仍可写成一个 $5\times5$ 可逆矩阵；求逆即可恢复最初五个整数，再按正逆位规则映射回塔罗牌名并拼接 flag。

## 解题过程

### 1. 把轮转写成矩阵

设初始向量为：

$$
\mathbf{x}=(a,b,c,d,e)^\mathsf{T}.
$$

一轮 `Fortune_wheel` 为：

```python
def Fortune_wheel(fate):
    return [fate[i] + fate[(i + 1) % 5] for i in range(5)]
```

所以：

$$
\begin{pmatrix}
a'\\b'\\c'\\d'\\e'
\end{pmatrix}=A
\begin{pmatrix}
a\\b\\c\\d\\e
\end{pmatrix},
\qquad
A=
\begin{pmatrix}
1&1&0&0&0\\
0&1&1&0&0\\
0&0&1&1&0\\
0&0&0&1&1\\
1&0&0&0&1
\end{pmatrix}.
$$

连续执行 250 次后：

$$
\mathbf{y}=A^{250}\mathbf{x}.
$$

`A` 是循环矩阵。也可以从特征值看出它可逆：五阶循环移位矩阵的特征值为五次单位根 $\omega^k$，因此 `A` 的特征值是 $1+\omega^k$。因为五阶是奇数，不存在 $\omega^k=-1$，所以没有零特征值。于是 $A^{250}$ 仍然可逆。

### 2. 生成 250 轮系数矩阵

官方 WP 用 Sage 在有理数域中保留符号变量，直接让第一行经历 250 轮，从中取得循环矩阵的系数：

```sage
R.<a, b, c, d, e> = PolynomialRing(QQ, 5)

def Fortune_wheel(fate):
    return [fate[i] + fate[(i + 1) % 5] for i in range(5)]

fate = [a, b, c, d, e]
for _ in range(250):
    fate = Fortune_wheel(fate)

# 明确按 a,b,c,d,e 的顺序取第一行系数，避免依赖 dict.values() 顺序
v = vector(QQ, [fate[0].monomial_coefficient(x) for x in (a, b, c, d, e)])
C = matrix(QQ, 5, 5, lambda i, j: v[(j - i) % 5])

assert C.rank() == 5
```

这里的 `C` 就是 $A^{250}$。与 PDF 原代码相比，显式按 `(a,b,c,d,e)` 提取系数更稳妥，不会把多项式字典的内部迭代顺序当成数学保证。

### 3. 对最终向量求逆

题目给出的最终值为：

```sage
final_fate = vector(ZZ, [
    2532951952066291774890498369114195917240794704918210520571067085311474675019,
    2532951952066291774890327666074100357898023013105443178881294700381509795270,
    2532951952066291774890554459287276604903130315859258544173068376967072335730,
    2532951952066291774890865328241532885391510162611534514014409174284299139015,
    2532951952066291774890830662608134156017946376309989934175833913921142609334,
])
```

按前面的列向量约定，直接计算：

```sage
initial_fate = C.inverse() * final_fate
print(initial_fate)
```

输出：

```text
(-19, -20, 20, -15, 41)
```

将这五个数重新执行 250 轮，得到的五个大整数与 PDF 逐项完全一致，因此矩阵方向和抄录值均已核验。

### 4. 映射回塔罗牌

牌表由 22 张大阿卡纳和 56 张小阿卡纳组成。负数表示逆位，题目使用：

```python
index = value ^ -1
```

在 Python 中 `x ^ -1 == ~x == -x - 1`，所以负数 `-1` 对应下标 0、`-2` 对应下标 1，以此类推。

完整映射和拼接代码如下：

```python
values = [-19, -20, 20, -15, 41]

major_arcana = [
    "The Fool", "The Magician", "The High Priestess", "The Empress",
    "The Emperor", "The Hierophant", "The Lovers", "The Chariot",
    "Strength", "The Hermit", "Wheel of Fortune", "Justice",
    "The Hanged Man", "Death", "Temperance", "The Devil",
    "The Tower", "The Star", "The Moon", "The Sun",
    "Judgement", "The World",
]

wands = [
    "Ace of Wands", "Two of Wands", "Three of Wands", "Four of Wands",
    "Five of Wands", "Six of Wands", "Seven of Wands", "Eight of Wands",
    "Nine of Wands", "Ten of Wands", "Page of Wands",
    "Knight of Wands", "Queen of Wands", "King of Wands",
]
cups = [
    "Ace of Cups", "Two of Cups", "Three of Cups", "Four of Cups",
    "Five of Cups", "Six of Cups", "Seven of Cups", "Eight of Cups",
    "Nine of Cups", "Ten of Cups", "Page of Cups",
    "Knight of Cups", "Queen of Cups", "King of Cups",
]
swords = [
    "Ace of Swords", "Two of Swords", "Three of Swords", "Four of Swords",
    "Five of Swords", "Six of Swords", "Seven of Swords", "Eight of Swords",
    "Nine of Swords", "Ten of Swords", "Page of Swords",
    "Knight of Swords", "Queen of Swords", "King of Swords",
]
pentacles = [
    "Ace of Pentacles", "Two of Pentacles", "Three of Pentacles",
    "Four of Pentacles", "Five of Pentacles", "Six of Pentacles",
    "Seven of Pentacles", "Eight of Pentacles", "Nine of Pentacles",
    "Ten of Pentacles", "Page of Pentacles", "Knight of Pentacles",
    "Queen of Pentacles", "King of Pentacles",
]

tarot = major_arcana + wands + cups + swords + pentacles

cards = []
for value in values:
    if value < 0:
        cards.append("re-" + tarot[value ^ -1])
    else:
        cards.append(tarot[value])

flag = ("hgame{" + "&".join(cards) + "}").replace(" ", "_")
print(flag)
```

五项分别为：

```text
-19 -> re-The Moon
-20 -> re-The Sun
 20 -> Judgement
-15 -> re-Temperance
 41 -> Six of Cups
```

最终得到：

```text
hgame{re-The_Moon&re-The_Sun&Judgement&re-Temperance&Six_of_Cups}
```

## 方法总结

本题的“大整数”只是可逆线性变换重复 250 次后的表象。把状态写成向量后，每轮都是固定循环矩阵 $A$，因此最终状态为 $A^{250}\mathbf{x}$；证明矩阵满秩、求逆、再恢复牌面即可。

实现时最容易出错的地方有三个：行向量与列向量方向、循环矩阵的旋转方向，以及负数逆位编码 `value ^ -1` 的含义。恢复后再正向执行 250 轮，与题目给出的五个数逐项比较，是成本最低且最可靠的验算。
