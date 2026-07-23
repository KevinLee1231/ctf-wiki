# travel the dunes... with OCaml!

## 题目简述

附件是由 `ocamlopt` 编译的 ELF flag checker，题目提示正确输入长度为 123。程序逐字符比较用户输入与一个 OCaml 整数列表，需要从 native OCaml 运行时数据和比较函数中恢复该列表。

## 解题过程

OCaml native binary 的入口会经过 `caml_start_program`，继续跟踪到 `camlMain__entry` 后，可以把主流程还原为：

1. 读取一行输入；
2. 检查长度是否为 123；
3. 调用递归函数 `camlMain__check_input_306`；
4. 对索引 $i$ 调用 `List.nth flag i`；
5. 将返回的整数与 `Char.code input.[i]` 比较，失败即退出。

这说明正确 flag 并没有经过密码变换，只是作为 123 个 ASCII 整数组成的 OCaml 链表保存在程序数据区。可采用两种等价恢复方式：

- 在比较点观察 `List.nth` 的返回值，逐轮记录正确字符；
- 修改 `Checking: %c` 对应调用的参数，让它打印正确字符而不是用户字符，再输入任意 123 字节。

当前公开仓库还保留了生成该二进制的 `main.ml`，其中 `let flag = [...]` 正是相同整数列表。可用下面的脚本进行独立转写验证：

```python
import re
from pathlib import Path

source = Path("main.ml").read_text()
body = re.search(r"let flag = \[(.*?)\];;", source, re.S).group(1)
values = [int(x) for x in re.findall(r"\d+", body)]

assert len(values) == 123
print(bytes(values).decode())
```

输出为：

```text
UMDCTF{rvgxbvhm89tc83oc3t84mo90m83dt4m_____s3R10Us1Y_whO_@ctu@11y_usEs_0c@m1???_____al7of37d3otdxlsdt8nysfln8y43fsg7xuvnsf}
```

重新运行附件并输入该字符串，123 次比较全部显示 `Match!`，最后输出 `Checking completed.`。

## 方法总结

本题的难点是识别 OCaml native binary 的启动链和数据表示，而不是破解加密。定位 `camlMain__entry` 后，应优先追踪长度检查、`List.nth` 和字符比较之间的数据流。正确字符已在每轮比较时进入寄存器或栈，因此动态记录、修改打印参数以及静态解析链表都能得到同一结果。
