# DownUnderCTF 2021 - bullet hell

## 题目简述

远程 shell 是一个 ncurses 弹幕游戏：玩家字符为 `p`，用 `h/j/k/l` 移动，每轮得一分，坚持到 1337 分即可获得 flag。界面中看不到弹幕，是因为绘制弹幕的语句被注释掉了；弹幕的位置更新和碰撞检测仍然照常执行。好在程序固定调用 `srand(0)`，所有弹幕都可在本地精确重放。

## 解题过程

### 恢复游戏状态与判定顺序

核心结构很简单：

```c
typedef struct {
    int x, y;
    int dx, dy;
} bullet_t;

typedef struct {
    int player_x, player_y;
    int score;
    bullet_t **bullets;
    size_t bullets_len;
} game_t;
```

每轮先在特定时刻生成弹幕，再调用 `update_bullets`。该函数先检查弹幕当前位置是否与玩家重合，随后才执行 `x += dx; y += dy`，并删除越界弹幕。之后程序读取一个按键、移动玩家、分数加一。因此模拟器必须严格保持这个顺序，差一轮就会与远程状态脱节。

弹幕实际不可见的原因是下面一行被注释：

```c
// mvwinsch(window, bullet->y, bullet->x, BULLET_CHAR);
```

但紧随其后的碰撞判断仍然有效。

### 精确模拟四种弹幕

主循环每 25 轮调用一次 `generate_bullets`。每次先执行 `rand() % 4` 选择类型，然后按相同顺序继续消耗 glibc 的 `rand()`：

| 类型 | 生成方式 |
| --- | --- |
| 0 | 从顶部三分之一的随机位置生成 35 枚向下弹幕 |
| 1 | 在左右边缘按行生成弹幕，分别向右下或左下移动 |
| 2 | 顶部几乎整行向下发射，只留下两个随机安全列 |
| 3 | 选择四个随机中心，各向八个方向发射一枚弹幕 |

程序入口固定执行：

```c
srand(0);
```

Python 可通过 `ctypes` 加载与目标相同的 glibc，从同一状态取得随机数：

```python
from ctypes import CDLL

libc = CDLL("libc.so.6")
libc.srand(0)

def rand():
    return libc.rand()
```

模拟器需要按二进制逐分支实现弹幕生成，连同看似无关的每一次 `rand()` 调用都不能省略。弹幕更新可概括为：

```python
for b in bullets:
    if (b.x, b.y) == (player_x, player_y):
        raise Collision
    b.x += b.dx
    b.y += b.dy
bullets = [b for b in bullets if 1 <= b.x <= 29 and 1 <= b.y <= 39]
```

### 生成并回放获胜按键

把本地模拟器改成显示弹幕后，可以手动选择安全移动，也可以让搜索程序在四个方向和原地等待之间寻找下一步。只要记录一条能存活 1337 轮的按键串，就可一次性粘贴到 SSH 会话；固定随机种子保证远程出现完全相同的弹幕序列。

下图是开启弹幕显示后的模拟过程。橙色弹幕的空间分布和青色玩家位置是理解、校验碰撞时序所必需的视觉信息：

![本地模拟器显示原程序隐藏弹幕并规避碰撞的动态过程](./DownUnderCTF2021-bullet-hell-wp/visible-bullet-simulation.gif)

到达 1337 分后得到：

```text
DUCTF{i_th1nk_w3're_g0nna_n33d_a_lunatic_d1fficul7y}
```

## 方法总结

本题的核心是确定性程序状态重放。应先恢复弹幕结构、生成分支和每轮执行顺序，再用相同 glibc、相同种子以及完全一致的 `rand()` 调用次数进行模拟。不可见只影响渲染，不影响状态；把本地弹幕可视化后生成安全按键串，再在远程重放即可稳定通关。
