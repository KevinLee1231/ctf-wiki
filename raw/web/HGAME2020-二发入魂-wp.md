# 二发入魂！

## 题目简述

服务端用 PHP 的 `mt_rand()` 生成随机数，并要求提交本轮使用的种子。接口允许一次取得指定数量的连续输出，因此可以利用 MT19937 状态转移的线性关系，仅凭两个特定位置的输出反推初始种子。

## 解题过程

先用同一 Session 访问首页，再请求：

```text
/random.php?times=228
```

响应是 228 个连续的 `mt_rand()` 结果。取第一个和最后一个输出，分别对应脚本中的 `data[0]` 与 `data[-1]`。MT19937 的输出会经过 temper 变换；逆向时先撤销右移异或、带掩码的左移异或等步骤，得到相关状态字，再利用 PHP 所用 twist 和初始化递推反推出种子。

官方解法直接调用 Ambionics 的 `mt_rand-reverse` 实现。该实现区分 PHP 版本使用的生成器 flavour，并把两个输出间的位置关系作为参数；本题调用形式为：

```python
import json
import requests

session = requests.Session()
base = "http://target"

session.get(base + "/index.php")
response = session.get(base + "/random.php?times=228")
values = json.loads(response.content)

# main 来自 mt_rand-reverse；最后两个 0 表示题目对应的偏移和 PHP flavour。
seed = main(values[0], values[-1], 0, 0)

result = session.post(base + "/verify.php", data={"ans": seed})
print(result.text)
```

恢复和提交必须共用一个 `requests.Session()`，否则新连接拿到另一份会话状态，正确种子也无法通过验证。完整逆 temper、twist 反演和 PHP 种子回溯代码可直接采用 [Ambionics 的 mt_rand-reverse](https://github.com/ambionics/mt_rand-reverse)；它不是一个在线“黑盒解密站”，而是本题所需算法的可审计实现。

## 方法总结

- 核心技巧：收集相隔固定位置的两个 `mt_rand()` 输出，逆 temper 与状态转移，再恢复种子。
- 关键细节：输出顺序、两输出间距、PHP flavour 和 Session 必须与目标完全一致。
- 复用要点：看到可批量获取 PRNG 连续输出且服务端把种子当秘密时，应检查生成器是否可逆；MT19937 只适合模拟，不具备密码学安全性。
