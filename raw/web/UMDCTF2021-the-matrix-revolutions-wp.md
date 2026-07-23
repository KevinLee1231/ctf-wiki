# The Matrix Revolutions

## 题目简述

第三关把 flag 拆成多个字符，分别放在编号路径中。单个页面只返回一个字符，需要枚举连续编号并按编号顺序拼接。

## 解题过程

访问根路径观察链接或响应规律后，依次请求 `/0` 到 `/20`：

```python
import requests

base = "http://target"
chars = []
for i in range(21):
    response = requests.get(f"{base}/{i}", timeout=5)
    response.raise_for_status()
    chars.append(response.text.strip())

print("".join(chars))
```

若响应包含 HTML，应只提取页面给出的单字符字段，避免把标签一起拼入。按数字而不是字符串字典序排列，最终得到：

```text
UMDCTF-{r0b0t5_43v3r}
```

## 方法总结

可预测对象编号会造成直接对象枚举，即使每个对象只泄露一小段信息，组合后仍可能暴露完整秘密。脚本应明确起止范围、检查状态码和单字符约束，并按整数顺序拼接。
