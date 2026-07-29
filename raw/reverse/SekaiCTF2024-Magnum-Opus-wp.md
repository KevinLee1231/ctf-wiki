# Magnum Opus

## 题目简述

附件看起来只有一行 `pickle.loads(...)`，其中嵌入了约 25 万字节的手工构造 Pickle 字节流。服务端使用 Python 3.11.9；连接后会连续给出 10 个编码过的数独，只有提交程序内部认可的结果才能读取 flag。

难点不在求普通数独，而在还原 Pickle 虚拟机实际执行的程序。它先求出标准答案，再用 glibc 的 `rand()` 随机改坏 11 个格子，比较时要求提交这个被修改后的网格。

## 解题过程

### 1. 识别 Pickle 中的可执行结构

Pickle 不是单纯的数据格式。`GLOBAL`、`STACK_GLOBAL`、`REDUCE` 等操作码可以取得任意 Python 对象并调用它。题目字节流大量使用 memo、无效果的 `MARK/POP`、重复压栈，以及字符逐个拼接来隐藏名称；同时动态调用 `types.CodeType`，把 Python 3.11 原始字节码组装成函数后交给 `exec`。

分析时可以先用 `pickletools.genops()` 顺序查看操作码，再重点跟踪以下对象：

```text
builtins.exec
types.CodeType
builtins.getattr
ctypes.CDLL
time.time
Crypto.Util.number.bytes_to_long / long_to_bytes
base64.b64decode / b64encode
sudokum.generate / sudokum.solve
```

载荷还利用 C 版 `_pickle` 与纯 Python 解析器对特殊 `INT` 文本的类型差异构造常量，使一些简单的替代解析方式在创建 `CodeType` 时失败。因此应以题目指定的 CPython 3.11.9 行为为准，不能只把它当成普通文本反序列化。

把动态 CodeType 的常量、名字表和字节码还原后，每轮核心逻辑等价于：

```python
while True:
    grid = sudokum.generate(mask_rate=0.56)
    solves = [sudokum.solve(grid)[1] for _ in range(10)]
    if all(x == solves[0] for x in solves):
        real_solve = solves[0]
        break

print(encode_grid(grid))

libc.srand(int(time()))
for _ in range(11):
    real_solve[libc.rand() % 9][libc.rand() % 9] = libc.rand() % 9 + 1

if input_grid != real_solve:
    exit(1)
```

这个单元在最终 Pickle 中重复 10 次，全部通过后才打开 `flag.txt`。

### 2. 解码题面并正常求数独

服务端把 9×9 网格拼成 81 位十进制整数，再执行 `long_to_bytes` 和 Base64。逆过程为：

```python
from base64 import b64decode
from Crypto.Util.number import bytes_to_long

raw = recv_line.strip()
digits = str(bytes_to_long(b64decode(raw))).zfill(81)
grid = [
    [int(digits[row + col]) for col in range(9)]
    for row in range(0, 81, 9)
]
```

使用与服务端相同的 `sudokum.solve(grid)[1]` 可得到标准解。这里必须保留前导零，所以解码后要调用 `zfill(81)`。

### 3. 同步 glibc rand 并重放 11 次破坏

随机数不是 Python 的 `random`，而是通过：

```python
CDLL('/lib/x86_64-linux-gnu/libc.so.6').rand()
```

生成。服务端在输出题面后立即执行 `srand(int(time()))`，种子精度只有一秒。客户端收到题面后立刻以本地当前 Unix 时间播种同一套 glibc PRNG，通常就能处于同一秒：

```python
from ctypes import CDLL
from time import time

libc = CDLL('/lib/x86_64-linux-gnu/libc.so.6')
libc.srand(int(time()))

answer = sudokum.solve(grid)[1]
for _ in range(11):
    row = libc.rand() % 9
    col = libc.rand() % 9
    value = libc.rand() % 9 + 1
    answer[row][col] = value
```

一次循环消耗三个 `rand()`，共 33 次。若连接恰好跨越整秒边界，种子会差 1；最稳妥的处理是尽量在读取题面后立即播种，失败时重连，调试阶段也可同时记录 `now - 1`、`now`、`now + 1` 三个候选结果确认时序。

### 4. 按原格式编码答案

服务端输入解码过程与题面相同，因此把被修改后的 81 个数字拼接成整数，再编码为字节和 Base64：

```python
from base64 import b64encode
from Crypto.Util.number import long_to_bytes

digits = ''.join(''.join(map(str, row)) for row in answer)
payload = b64encode(long_to_bytes(int(digits)))
sendline(payload)
```

完整客户端重复上述流程 10 次。每轮都要重新接收网格、以当时秒数播种、求解并重放 11 次修改；全部比较相等后，服务端打印 `Good job! Here is your flag:` 和 flag。

仓库中服务端使用的 flag 为：

```text
SEKAI{when_you_implement_the_same_VM_3_times_there's_bound_to_be_discrepancies}
```

## 方法总结

题目用 Pickle 栈机、memo 字符表、伪操作和动态 `CodeType` 隐藏了一段并不复杂的数独协议。逆向时不必逐字节美化整个 25 万字节载荷，而应围绕产生副作用的 `GLOBAL/REDUCE/exec` 建立数据流，恢复导入、随机数播种、网格修改与比较逻辑。

真正容易遗漏的是两处实现细节：随机源是 glibc `rand()` 而非 Python PRNG，种子是秒级 `int(time())`；服务端需要的是被随机改坏的答案，而不是合法数独解。正确复现这两点即可稳定完成 10 轮交互。
