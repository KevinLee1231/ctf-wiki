# 3v3ntl0g

## 题目简述

附件 `shuffled.bin` 是从 Linux 键盘输入设备读取的原始事件，但记录顺序被打乱。题目还提示目标使用法语 AZERTY 键盘，需要按时间恢复事件并将物理键码映射为实际字符。

## 解题过程

64 位 Linux 上的 `input_event` 结构由两个 64 位时间字段、两个 16 位字段和一个 32 位值组成，总长 24 字节：

```c
struct input_event {
    struct timeval time;
    __u16 type;
    __u16 code;
    __s32 value;
};
```

对应的小端解析格式为 `<qqHHi`。其中：

- `type == 1` 表示 `EV_KEY`；
- `value == 1` 表示按下；
- `value == 0` 表示释放；
- `value == 2` 表示按住后的重复事件；
- `code` 是 Linux 输入子系统定义的物理键码。

先按 `(秒, 微秒)` 对所有 24 字节记录排序，再维护 Shift、AltGr 和 Ctrl 的按下状态。Linux 键码描述的是键位，不能直接套用 QWERTY 字符：例如 AZERTY 的键码 16 对应 `A`，键码 30 对应 `Q`；数字行还需要按 Shift 状态选择数字或符号。下面的脚本包含本次数据实际使用到的 AZERTY 映射：

```python
import struct

LETTERS = {
    16: "a", 17: "z", 18: "e", 19: "r", 20: "t",
    21: "y", 22: "u", 23: "i", 24: "o", 25: "p",
    30: "q", 31: "s", 32: "d", 33: "f", 34: "g",
    35: "h", 36: "j", 37: "k", 38: "l", 39: "m",
    44: "w", 45: "x", 46: "c", 47: "v", 48: "b",
    49: "n",
}

NUMBER_ROW = {
    2: ("&", "1"),
    3: ("é", "2"),
    4: ('"', "3"),
    5: ("'", "4"),
    6: ("(", "5"),
    7: ("-", "6"),
    8: ("è", "7"),
    9: ("_", "8"),
    10: ("ç", "9"),
    11: ("à", "0"),
}

data = open("shuffled.bin", "rb").read()
events = sorted(
    struct.iter_unpack("<qqHHi", data),
    key=lambda event: (event[0], event[1]),
)

shift = False
altgr = False
ctrl = False
plain = []

for _, _, event_type, code, value in events:
    if event_type != 1:
        continue

    if code == 42:
        shift = value != 0
        continue
    if code == 100:
        altgr = value != 0
        continue
    if code == 29:
        ctrl = value != 0
        continue
    if value != 1:
        continue
    if ctrl and code == 46:
        break

    if altgr and code == 5:
        plain.append("{")
    elif altgr and code == 13:
        plain.append("}")
    elif code in LETTERS:
        char = LETTERS[code]
        plain.append(char.upper() if shift else char)
    elif code in NUMBER_ROW:
        plain.append(NUMBER_ROW[code][int(shift)])
    elif code == 50:
        plain.append("?" if shift else ",")

print("".join(plain))
```

输出为：

```text
N0PS{c4n_y0U_R34d_Th15??}
```

## 方法总结

恢复输入设备数据需要同时解决三层问题：按 ABI 正确切分结构、用时间戳还原顺序、按键盘布局和修饰键状态解释键码。只列出“按下了哪个键”还不够，AZERTY 的字母位置、数字行和 AltGr 组合都会改变最终字符。显式使用 `<qqHHi` 也比依赖平台原生 `long` 大小更可靠。
