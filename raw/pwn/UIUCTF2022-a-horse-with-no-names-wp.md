# A Horse with No Names

## 题目简述

题目是一个 Python 表达式 jail。输入依次经过三层限制：开头不能出现 4 个连续 ASCII 字母；全文最多只能使用 4 种不同的非单词字符；最后用 `compile(..., "eval")` 编译，并把最外层代码对象的 `co_names` 清空后交给 `eval`。返回值会被转成列表并随机打乱。

```python
if re.match(r"[a-zA-Z]{4}", horse):
    ...
elif len(set(re.findall(r"[\W]", horse))) > 4:
    ...
else:
    code = compile(horse, "<horse>", "eval").replace(co_names=())
    discovery = list(eval(code))
    random.shuffle(discovery)
```

flag 位于 `/flag.txt`。决定性漏洞是 `CodeType.replace(co_names=())` 只修改外层代码对象，不会递归清理生成器表达式所包含的内层代码对象。第一版的 `re.match` 又只检查字符串开头；Python 的 Unicode 标识符规范化则提供了一条同时兼容本题与修复版的绕过路线。

## 解题过程

### 绕过字符过滤

官方 payload 使用数学斜体 Unicode 字母拼出 `eval` 和 `input`：

```python
((𝘦𝘷𝘢𝘭(𝘪𝘯𝘱𝘶𝘵()) for x in ((),)))
```

这些字符在 Python 标识符解析阶段会规范化为普通的 `eval`、`input`，但不会命中 `[a-zA-Z]`。输入以左括号开头，原题的 `re.match` 本来就不会检查后续文本，所以在这一版中把 `eval`、`input` 写成普通 ASCII 也能绕过第一项限制；使用数学斜体字母的价值在于同一 payload 还能通过下一题改用 `re.search` 的过滤。整条表达式只使用 `(`、`)`、空格和逗号四类非单词字符，恰好不超过限制。

### 利用嵌套代码对象

编译生成器表达式时，最外层代码只负责创建生成器；生成器主体会成为 `co_consts` 中的另一个代码对象。题目只对外层调用 `.replace(co_names=())`，内层仍保留对 `eval` 和 `input` 的名称引用。

外层 `eval` 返回生成器后，程序执行 `list(...)`，这会真正推进生成器并调用内层的 `input()`，形成完全未经过前述过滤的第二阶段输入。第二行发送：

```python
print(open('/flag.txt').read()) or ()
```

`print` 的副作用先把 flag 输出；表达式随后返回空元组，使生成器只产生一个低价值元素。即使程序再随机打乱列表，也不会影响已经打印到标准输出的 flag。

最终得到：

```text
uiuctf{my_challenges_have_abandoned_any_pretense_of_practical_applicability_and_im_okay_with_that}
```

## 方法总结

- 核心技巧：利用 `re.match` 的起始位置盲区或 Unicode 兼容字符绕过 ASCII 正则，并把危险名称放进生成器的内层代码对象，避开只修改外层 `co_names` 的过滤。
- 识别信号：输入经 `compile` 后只调用一次 `CodeType.replace`，同时允许 lambda、生成器或推导式等会产生嵌套代码对象的表达式。
- 复用要点：审计 Python 字符过滤时要比较“过滤器看到的文本”和“解释器规范化后的标识符”；清理代码对象必须递归处理所有嵌套 `CodeType`，而且即便返回值被打乱，`print`、异常等副作用仍可作为输出通道。
