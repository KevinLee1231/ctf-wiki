# JSON ⊂ YAML?

## 题目简述

程序把同一行输入分别交给 Python `json.loads`、PyYAML 6.0.1 的 `safe_load`（YAML 1.1 解析行为）和 ruamel.yaml 0.17.40 的 `safe_load`（YAML 1.2）。第一问要求输入是合法 JSON，但 JSON 与 YAML 1.1 的解析结果不相等；第二问要求 JSON 能解析而 YAML 1.2 报错。

核心判定可概括为：

```python
as_json = json.loads(s)
as_yaml_1_1 = yaml.safe_load(s)
if as_json != as_yaml_1_1:
    print(flag1)

try:
    ruamel.yaml.safe_load(s)
except Exception:
    print(flag2)
```

题目虽然以交互程序承载，但决定性障碍是 JSON 与 YAML 的表示及解析语义差异，因此归入 `crypto` 的表示层问题。

## 解题过程

### 第一问：指数数字的类型差异

JSON 数字允许没有小数点的指数形式，也允许指数部分不带显式正负号。例如：

```text
1e2
```

`json.loads` 将它解析为数值 `100.0`。PyYAML 所采用的 YAML 1.1 浮点解析规则要求十进制浮点数带小数点，指数部分还要求显式符号，因此 `1e2` 不匹配其浮点类型，而被解析为字符串。两边结果分别是数值与字符串，比较不相等，得到第一问 flag。

### 第二问：重复映射键

JSON 语法允许对象成员名重复，尽管 RFC 8259 建议名称应唯一；YAML 1.2 则规定 mapping 的键必须唯一。输入：

```json
{"":0,"":0}
```

Python JSON 解析器接受该对象，后出现的同名键覆盖先前值；ruamel.yaml 检测到重复键并抛出异常，于是进入第二问的 `except` 分支。该输入也能被题目使用的 YAML 1.1 解析器正常处理，满足题目的隐含组合条件。

还可以把两种差异合并到一个 13 字符输入中：

```json
{"":0,"":1e2}
```

重复键触发 YAML 1.2 异常，而 YAML 1.1 对 `1e2` 的字符串解释又与 JSON 的数值解释不同，因此一次输入可以同时满足两问。

## 方法总结

- 核心技巧：寻找同一文本在不同格式规范或解析器版本中的类型、合法性差异。
- 识别信号：同一输入依次经过 JSON、YAML 1.1 和 YAML 1.2，并直接比较解析后的对象或异常状态。
- 复用要点：不要只看“格式 A 是格式 B 的子集”这类概括；还要核对数字 schema、重复键规则以及具体库采用的版本和扩展行为。
