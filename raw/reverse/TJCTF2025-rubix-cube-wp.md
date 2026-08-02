# rubix-cube

## 题目简述

生成器先对魔方执行 1000 次未知随机转动，把此时 54 个贴纸的颜色编码成 flag；随后固定 `random.seed(42)`，再执行 $20\times50=1000$ 次转动，并只发布最终状态 `cube_scrambled.txt`。无需求解真实魔方，只要逆放固定 seed 生成的第二段操作，就能回到 flag 被编码时的状态。

## 解题过程

第二段操作使用的动作表为：

```python
moves = ["U", "L", "F", "B'", "D'", "R'"]
```

每次索引由 `random.randint(0, 5)` 产生。用相同 seed 重建 1000 个索引，再按倒序执行对应逆动作即可。题目说“忘了写几个方法”，但不必补全六个逆转函数：每个动作都是四分之一圈，连续执行四次回到原状，所以动作 $M$ 的逆就是 $M^3$。

为避免导入 `rubixcube.py` 时执行文件底部的生成代码并覆盖附件，应把 `RubiksCube` 类单独复制到求解脚本，或先给生成部分加上 `if __name__ == "__main__":` 保护。核心恢复代码如下：

```python
import ast
import random

# RubiksCube 类沿用题目源码；本文件只保留类定义，不执行生成部分。
cube = RubiksCube()
with open("cube_scrambled.txt", encoding="utf-8") as f:
    cube.faces = ast.literal_eval(f.read())

moves = ["U", "L", "F", "B'", "D'", "R'"]
rng = random.Random(42)
order = [rng.randint(0, 5) for _ in range(1000)]

for index in reversed(order):
    # 同一个四分之一圈做三次，等价于做一次逆转。
    for _ in range(3):
        cube.apply_moves(moves[index])

flag = "tjctf{" + cube.get_flat_cube_encoded() + "}"
print(flag)
```

`get_flat_cube_encoded` 从魔方字典的字符串表示中筛出 Unicode 码点大于 256 的贴纸 emoji，并把每个码点映射为 `chr(codepoint % 94 + 33)`。恢复出的 54 个字符按原函数编码后得到：

```text
tjctf{G>BGG@BBGA>B>@B??>@G?@B??B>>?GA>@G@ABB@A?AA?@?AA>AG>G@}
```

仓库官方脚本打印 flag 后又显示 `False`，原因只是 `flag.txt` 末尾带换行而脚本未调用 `.strip()`；两边去除行尾后完全相等。上述“三次同向转动”实现也已在发布状态上独立复验为 `True`。

## 方法总结

- 核心技巧：重放固定 PRNG seed 生成的动作序列，并以逆序、逆动作撤销状态变换。
- 识别信号：先生成秘密状态、再固定 seed 扰动、最后只发布扰动结果，是典型的可逆状态回放题。
- 复用要点：阶为 4 的转动满足 $M^{-1}=M^3$；验证文本结果时要先规范化无语义的行尾，避免把换行差异误判为算法失败。
