# artifact

## 题目简述

题目给出一组浮点数 `points.txt` 以及生成程序。程序先把每个 flag 字节 $b$ 映射为圆上的纵坐标

$$
x_b=\sqrt{r^2-(31b)^2},\qquad r=\frac{24901}{2\pi},
$$

随后根据一段百万次空循环的实际耗时，对当前已有的全部点施加一次角度旋转。最终文件没有记录每轮耗时，因此需要利用 flag 的可打印字符范围和已知结尾 `}`，从最后一轮开始逐步消去旋转。

## 解题过程

生成器的 `add_spin` 并不是直接给数值加常量，而是先把纵坐标还原为角度，再增加本轮角度：

$$
\theta=\arcsin(x/r),\qquad
x'=r\sin\left(\theta+\frac{360\cdot1000t}{24901}\right).
$$

新加入的最后一个点只经历最后一轮旋转。TJCTF flag 固定以 `}` 结束，所以将最后一个观测值的角度减去字符 `}` 对应的基准角度，就得到最后一轮旋转量。消去这次旋转后，倒数第二个点便只剩它加入时那一轮及更早已消去的影响；继续从右向左，在 ASCII 候选中寻找角度最接近的字节，并把该轮角度从所有更早的点中扣除。

下面保留官方脚本的核心逻辑。候选限制到 ASCII 的 `0..127`，同时要求观测角与候选角的差接近当前估计的轮转量，从而抑制浮点误差造成的误判。

```python
from math import asin, degrees, pi, radians, sin, sqrt
from ast import literal_eval

circ = 24901
r = circ / (2 * pi)
points = literal_eval(open("points.txt", encoding="utf-8").read())
bases = [sqrt(r * r - (31 * b) ** 2) for b in range(128)]

def angle(x):
    return degrees(asin(x / r))

def unspin(values, delta):
    return [r * sin(radians(angle(x) - delta)) for x in values]

# 最后一字节已知为 `}`，先估计最后一轮的角度。
turn = angle(points[-1]) - angle(bases[ord("}")])
flag = ""

for pos in range(len(points) - 1, -1, -1):
    candidates = []
    for b, base in enumerate(bases):
        delta = angle(points[pos]) - angle(base)
        if delta > 0 and delta > turn - 0.1:
            candidates.append((abs(delta - turn), b, delta))
    _, byte, recovered_turn = min(candidates)
    flag = chr(byte) + flag
    points = unspin(points, recovered_turn)
    turn = recovered_turn

print(flag)
```

仓库官方求解器在发布数据上输出：

```text
tjctf{t1me_sure_1s_sl0w_p12oasjui7}
```

## 方法总结

- 核心技巧：把非线性的正弦坐标重新变成角度，并按生成顺序的逆序逐轮消去公共旋转。
- 识别信号：每加入一个新元素就变换整个前缀，意味着最后一个元素受到的变换最少，适合从尾部反推。
- 复用要点：浮点数据不应直接做相等比较；应限制字符集、利用已知格式锚点，并在角度空间设置合理容差。
