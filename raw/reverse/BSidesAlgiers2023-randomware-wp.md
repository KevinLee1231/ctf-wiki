# RaN0dmWaR3

## 题目简述

附件包含多层混淆的 `cool_code.py`、未剥离符号的 Go 共享库 `lib.so`，以及被打乱的 `files/file.png`。Python 外壳最终只负责加载共享库并调用导出函数；真正的文件变换位于 Go 代码中。

程序不是加密每个字节，而是用固定种子初始化 Go 的 `math/rand`，再调用 `Shuffle` 原地交换文件字节。置换不丢失信息，只要重现相同的交换序列并按相反顺序执行，就能完整恢复原文件。

## 解题过程

最外层 Python 代码先用 Base64、LZMA、`compile` 和 `exec` 包裹下一层。分析时不应直接执行未知 payload；可以从 AST 中寻找 XZ magic，只对匹配的常量做静态解码：

```python
import ast
import base64
import binascii
import lzma
from pathlib import Path

source = Path("cool_code.py").read_text(encoding="utf-8")
magic = b"\xfd7zXZ\x00"

while True:
    tree = ast.parse(source)
    candidates = []

    for node in ast.walk(tree):
        if not (
            isinstance(node, ast.Constant)
            and isinstance(node.value, bytes)
        ):
            continue

        raw = node.value
        if raw.startswith(magic):
            candidates.append(raw)
            continue

        try:
            decoded = base64.b64decode(raw, validate=True)
        except (binascii.Error, ValueError):
            continue
        if decoded.startswith(magic):
            candidates.append(decoded)

    if not candidates:
        break
    source = lzma.decompress(max(candidates, key=len)).decode()

print(source)
```

直接的 XZ 层之后还有 ROT13、Base64 和 `marshal` 组合层。该层把四段字符串按“第一段 ROT13 解码、第三段反转”的方式拼接，再 Base64 解码为 Python code object。可以只查看 code object 元数据，不调用其中的 `exec`：

```python
import ast
import base64
import codecs
import marshal

tree = ast.parse(source)
values = {
    node.targets[0].id: node.value.value
    for node in tree.body
    if isinstance(node, ast.Assign)
    and len(node.targets) == 1
    and isinstance(node.targets[0], ast.Name)
    and isinstance(node.value, ast.Constant)
    and isinstance(node.value.value, str)
}

payload = (
    codecs.decode(values["____"], "rot13")
    + values["_____"]
    + values["______"][::-1]
    + values["_______"]
)
code = marshal.loads(base64.b64decode(payload))
print(code.co_names)
print(code.co_consts)
```

若 `co_consts` 中仍有 Base64 编码的 XZ 数据，就继续解压该常量并重复上述检查。最终 code object 的名字表包含 `ctypes`、`cdll`、`LoadLibrary` 和 `cool_function`，常量表包含 `./lib.so`。因此无需执行混淆代码，也能确认核心 Python 逻辑等价于：

```python
import ctypes

library = ctypes.cdll.LoadLibrary("./lib.so")
library.cool_function()
```

`lib.so` 保留了 Go 符号，可用 `go tool nm` 定位 `cool_function` 与 `shuffleFile`，再用 `go tool objdump` 查看调用关系。逆向结果表明，`cool_function` 遍历 `./files`，而 `shuffleFile` 对每个文件执行以下操作：

1. 读入完整字节数组；
2. 用常量种子 `0x7331` 创建 `math/rand` 随机源；
3. 调用 `Rand.Shuffle`，回调中交换两个字节；
4. 把置换后的数组写回原文件。

若加密阶段的交换依次为 $S_1,S_2,\ldots,S_n$，恢复阶段必须执行 $S_n,\ldots,S_2,S_1$。下面的 Go 程序让 `Shuffle` 只记录交换对，不立即修改数据，随后倒序交换被破坏的文件：

```go
package main

import (
	"fmt"
	"math/rand"
	"os"
)

type swap struct {
	i int
	j int
}

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintf(os.Stderr, "usage: %s encrypted.png recovered.png\n", os.Args[0])
		os.Exit(2)
	}

	data, err := os.ReadFile(os.Args[1])
	if err != nil {
		panic(err)
	}

	rng := rand.New(rand.NewSource(0x7331))
	swaps := make([]swap, 0, len(data))
	rng.Shuffle(len(data), func(i, j int) {
		swaps = append(swaps, swap{i: i, j: j})
	})

	for k := len(swaps) - 1; k >= 0; k-- {
		s := swaps[k]
		data[s.i], data[s.j] = data[s.j], data[s.i]
	}

	if err := os.WriteFile(os.Args[2], data, 0o644); err != nil {
		panic(err)
	}
}
```

编译运行：

```bash
go run recover.go challenge/files/file.png recovered.png
file recovered.png
```

恢复后的 PNG 能通过文件格式检查，画面内容只是以下文本：

```text
shellmates{C00l_R4nD0mWarE}
```

由于原图仅承载可直接转写的单行文本，没有独立视觉证据价值，本题解不再保留这张截图。

## 方法总结

本题的关键是区分“随机”与“不可逆”。使用固定种子的伪随机数生成器会产生完全确定的交换序列，而 Fisher-Yates Shuffle 本身是若干可逆交换的组合。重放相同序列不等于解密；必须把交换顺序反过来，才能得到逆置换。

多层 Python 混淆只是加载器外壳，静态解包到 `ctypes` 调用后就应转向共享库。Go 二进制保留的函数名、类型信息和标准库调用路径能显著缩小分析范围。若要真正保护文件，不能用固定种子的字节置换代替加密，而应采用经过认证的加密算法并为每个文件使用不可预测的 nonce。
