# Keep Reversing and Nobody Explodes

## 题目简述

WelcomeCTF2021 的 Keep Reversing and Nobody Explodes 是浏览器端拆弹游戏。每局生成新的序列号，三类模块的正确操作由 WebAssembly 逻辑决定；答错会立刻爆炸。目标是逆向 WASM，而不是靠反复试错穷举随机实例。

## 解题过程

先查看主 JavaScript，定位生成序列号和调用 WASM 校验函数的位置。序列号格式为：两位 `A` 到 `F` 的字母、四位十进制数字、两位 `A` 到 `F` 的字母。

### 线缆模块

把 WASM 中的分支整理后，规则如下：

- 恰有 1 根黑线：剪黑线；
- 没有黑线且共 3 根：剪最后一根；
- 没有黑线且共 4 根：若第二根为红色则剪第一根；否则颜色种类不少于 3 时，按源码的零基下标 `颜色种类数 - 1` 剪线，其余情况剪第二根；
- 没有黑线且共 5 根：若存在绿线，剪最后一根绿线的前一根并循环回绕；否则剪第四根；
- 至少 2 根黑线且共 3 或 4 根：按零基下标 `非黑线数量` 剪线；
- 至少 2 根黑线且共 5 根：剪第四根。

### 序列按钮

取序列号中四位数字为 $k$。正确按键顺序是 `ABCD` 的全部 24 个排列按字典序排列后的第 $k\bmod24$ 项，`ABCD` 的编号为 0。

```python
from itertools import permutations

orders = ["".join(item) for item in permutations("ABCD")]
answer = orders[number % 24]
```

### 大按钮

取序列号中的四个字母 $p,q,r,s$，按 `A=0`、`B=1`、……、`F=5` 转为整数，计算：

$$
t=p\oplus q\oplus r\oplus s.
$$

当倒计时任意一位出现数字 $t$ 时按下按钮。

依次按上述规则完成三个模块即可得到：

```text
greyhats{Wh0_n33D5_a_p4rTnER}
```

## 方法总结

这道题的工作量在于把 WASM 控制流恢复成可执行规则表。主 JavaScript 用来确定参数含义和模块入口，WASM 用来确认真正判定逻辑。把每个分支整理成互斥条件后，就能对任意随机序列号稳定求解，而不必依赖某一局的固定答案。
