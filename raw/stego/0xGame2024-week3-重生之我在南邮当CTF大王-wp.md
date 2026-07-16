# 重生之我在南邮当CTF大王

## 题目简述

附件是一款 RPG 游戏，四段 flag 分散在地图事件 JSON 中；正常游玩可以取得其中几段，但直接检查 `data` 目录更完整。源仓库仅保留[游戏附件网盘入口](https://pan.quark.cn/s/68efc95ec2af)，下面已把四处数据、编码类型和拼接结果写入正文。

## 解题过程

RPG Maker 的地图和事件通常保存在 `data/Map*.json`。其中事件命令 `code: 401` 表示一行对话文本，因此可以在全部 JSON 中搜索 `flag1`、`flag2`、`flag3`，以及大量由 `呜`、`嗷`、`啊`、`~` 组成的异常字符串。

### 1. flag1

第一段藏在事件对象的 `name` 字段中：

```json
{"id": 5, "name": "MHhHYW1le05KVVBUXw==", "note": "flag1"}
```

Base64 解码：

```python
from base64 import b64decode

print(b64decode("MHhHYW1le05KVVBUXw==").decode())
```

得到：

```text
0xGame{NJUPT_
```

### 2. flag2 与 flag3

另外两张地图的对话参数直接写出了片段，无需二次解码：

```text
flag2:Has_
flag3:VerY_v3Ry_V3ry_
```

因此中间两段分别为：

```text
Has_
VerY_v3Ry_V3ry_
```

### 3. flag4

最后一张地图反复出现只由 `~呜嗷啊` 等字符组成的长字符串。这是“兽音编码”，其字符集合承担类似四进制符号表的作用。将完整字符串交给兼容的兽音译码器，结果为：

```text
YummY_FooD}
```

按 `flag1 → flag2 → flag3 → flag4` 的顺序拼接：

```text
0xGame{NJUPT_Has_VerY_v3Ry_V3ry_YummY_FooD}
```

## 方法总结

游戏类隐写题不应只沿正常剧情推进，还要检查资源目录、地图 JSON 和事件参数。RPG Maker 的事件数据是结构化文本，搜索 `flag`、备注字段和异常字符集通常能快速定位隐藏片段；找到片段后再分别判断 Base64、明文或自定义字符编码，最后严格按编号拼接并保留大小写。
