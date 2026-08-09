# CookieClicker

## 题目简述

游戏要求累计一百万块饼干，但设置页提示存在自动存档。程序目录中的 `save.xlsx` 保存了游戏状态，因此无需持续点击，关键是识别并修改存档字段。

## 解题过程

打开 `save.xlsx` 后可以看到前两个单元格分别对应金币和饼干数。把第二个单元格改为 `10000000`，保存后重新进入游戏，即可触发达标分支并看到：

```text
b'bjAwYnp7WXVtbXlDMDBraWV6fQ=='
```

去掉 Python 字节串外壳，对中间内容进行 Base64 解码：

```python
import base64

print(base64.b64decode("bjAwYnp7WXVtbXlDMDBraWV6fQ==").decode())
```

得到 flag：

```text
n00bz{YummyC00kiez}
```

## 方法总结

本题的决定性线索是自动存档，而不是点击速度。面对带本地状态的游戏题，应优先检查存档格式和字段语义；触发输出后还要继续识别其表示层编码。
