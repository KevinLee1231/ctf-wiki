# vr-keylog

## 题目简述

题目提供两次 VR 控制器轨迹记录。`source.txt` 同时含时间戳、控制器位置、四元数姿态和已知按键标签，可作为虚拟键盘标定样本；`unknown.txt` 只有另一会话中的位置与姿态。虚拟键盘位于控制器前向 $z$ 轴 2 个单位处，而且两次会话的观察坐标系存在整体旋转。目标是先对齐会话，再把未知轨迹点映射到最近的已知按键位置。

## 解题过程

先从标定记录中取已知字符 `t` 按下时的姿态 $q_s$，并把未知记录首个样本姿态记为 $q_u$。官方数据以这两个样本作为同一参考键，对齐旋转为

$$q_a=(q_uq_s^{-1})^{-1}.$$

对未知会话的每个位置和姿态分别应用 $q_a$，即可映射回标定坐标系。下面是核心对齐片段；其中 `q_unknown_anchor`、`q_source_t`、`unknown_positions` 和 `unknown_orientations` 均由两个 CSV 记录按时间戳解析得到：

```python
import numpy as np

def q_inverse(q):
    x, y, z, w = q
    norm2 = x*x + y*y + z*z + w*w
    return np.array([-x, -y, -z, w]) / norm2

def q_multiply(a, b):
    ax, ay, az, aw = a
    bx, by, bz, bw = b
    return np.array([
        aw*bx + ax*bw + ay*bz - az*by,
        aw*by - ax*bz + ay*bw + az*bx,
        aw*bz + ax*by - ay*bx + az*bw,
        aw*bw - ax*bx - ay*by - az*bz,
    ])

def q_matrix(q):
    x, y, z, w = q / np.linalg.norm(q)
    return np.array([
        [1-2*y*y-2*z*z, 2*x*y-2*z*w, 2*x*z+2*y*w],
        [2*x*y+2*z*w, 1-2*x*x-2*z*z, 2*y*z-2*x*w],
        [2*x*z-2*y*w, 2*y*z+2*x*w, 1-2*x*x-2*y*y],
    ])

align_q = q_inverse(q_multiply(q_unknown_anchor, q_inverse(q_source_t)))
align_rotation = q_matrix(align_q)
aligned_positions = [align_rotation @ p for p in unknown_positions]
aligned_orientations = [q_multiply(align_q, q) for q in unknown_orientations]
```

对齐后，根据提示把控制器位置向前投影 2 个单位，得到其在虚拟键盘平面上的光标点。标定会话中，每个已知按键时间戳都对应一个光标位置；未知会话的每个点选择欧氏距离最近的标定按键：

```python
source_key_points = np.array(source_key_points)
guessed = []
for point in unknown_key_points:
    index = np.argmin(np.linalg.norm(source_key_points - point, axis=1))
    guessed.append(source_key_labels[index])
```

最后按题目提示解释空格：单个空格替换为下划线，连续两个空格依次替换为左、右花括号。官方脚本对附件执行后输出：

```text
tjctf{i_have_friends_everywhere}
```

## 方法总结

- 核心技巧：用带标签的 VR 轨迹标定虚拟键盘，通过四元数求会话间旋转，再做最近邻按键分类。
- 识别信号：同一设备的两组姿态/位置日志、已知标签会话、固定的键盘前向偏移和不同视角提示。
- 复用要点：必须先做坐标系对齐再比较位置；时间戳用于把按键事件绑定到最近姿态样本，单纯比较原始坐标会因会话视角不同而失效。
