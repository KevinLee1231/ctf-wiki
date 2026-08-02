# garfield-mondays

## 题目简述

题目提供 Android APK 和登录网站。应用会检查星期与当前时间，生成网站账号密码；题面“Garfield hates Mondays”和官方提示都指向把设备日期设为星期一。真正需要逆向的是 APK 中对 `HH:mm` 的 SHA-256 检查、数字运算和字符映射，而服务器端使用参数化 SQLite 查询，没有 SQL 注入点。

## 解题过程

用 JADX 搜索硬编码 SHA-256 字符串，得到目标摘要：

```text
cf4627b3786c8bad8cb855567bda362d8eca1809ea8839423682715cdf3aadad
```

时间只有 $24\times60=1440$ 种，枚举 `00:00` 到 `23:59` 后唯一命中 `14:25`。应用去掉冒号得到整数 1425，再计算：

$$
1425\times100+225390=367890.
$$

它把 `1425` 与 `367890` 拼成 `1425367890`，再按 APK 内置映射逐字符替换：

```text
1→g, 4→f, 2→i, 5→e, 3→l,
6→d, 7→1, 8→2, 9→3, 0→4
```

所以密码是 `gfield1234`，用户名是应用中硬编码的 `garfield`。可直接复现客户端逻辑并提交：

```python
from hashlib import sha256
import requests

target = "cf4627b3786c8bad8cb855567bda362d8eca1809ea8839423682715cdf3aadad"
time = next(f"{h:02d}:{m:02d}" for h in range(24) for m in range(60)
            if sha256(f"{h:02d}:{m:02d}".encode()).hexdigest() == target)

digits = time.replace(":", "")
raw = digits + str(int(digits) * 100 + 225390)
table = str.maketrans("1425367890", "gfield1234")
password = raw.translate(table)

response = requests.post(
    "https://garfield-mondays.tjc.tf/form_handler",
    data={"username": "garfield", "password": password},
)
print(response.json()["message"])
```

比赛时也可把模拟器日期设为星期一、时间设为 14:25，点击应用按钮后从 logcat 观察生成值。服务返回：

```text
tjctf{g4rf1eld_lasagna_m0nday}
```

## 方法总结

- Android 时间门控不需要等待真实日期；模拟器可改时钟，静态分析也可直接枚举有限时间空间。
- 服务器使用占位参数查询，不能把“登录框”自动等同于 SQL 注入；正确链路是逆向官方客户端凭据算法。
- 哈希只隐藏 1440 种可能时间，不提供实际安全性。命中时间后应逐步保留去冒号、整数计算和字符替换，避免把最终密码写成无来源常量。
