# Level 24 Pacman

## 题目简述

题目是纯前端 Pac-Man 页面。游戏结束时会显示一串 Base64 文本，但第一次看到的字符串解码后只是故意设置的假 flag；题干中“收集一万枚金币”的提示用于引导检查前端 `index.js` 中与分数/礼物有关的逻辑，从源码找到第二串编码数据。

决定解法的是浏览器端源码与游戏状态，而不是服务端漏洞，因此归入 `web`。

## 解题过程

游戏失败后得到第一串：

```text
aGFlcGFpZW1rc3ByZXRnbXtydGNfYWVfZWZjfQ==
```

Base64 解码后为：

```text
haepaiemkspretgm{rtc_ae_efc}
```

字符串含有被打乱的 flag 结构。把它按两栏栅栏密码解码，可得：

```text
hgame{pratice_makes_perfect}
```

该值提交会被拒绝，说明它只是诱饵。继续检查 `index.js`，围绕 `score`、`10000`、`gift` 等字符串定位游戏结束与奖励分支，可以找到第二串：

```text
aGFldTRlcGNhXzR0cmdte19yX2Ftbm1zZX0=
```

这串数据在官方 PDF 中只出现在截图里；[同期公开题解](https://astralprisma.github.io/2025/02/17/hgame_25/)也保存了相同字符串。重复“Base64 → 两栏栅栏”两步，得到真正的 flag：

```text
hgame{u_4re_pacman_m4ster}
```

下面的短脚本可同时验证两组数据：

```python
import base64


def rail_fence_2_decode(text: str) -> str:
    split = (len(text) + 1) // 2
    upper, lower = text[:split], text[split:]
    return "".join(
        upper[i] + (lower[i] if i < len(lower) else "")
        for i in range(len(upper))
    )


samples = [
    "aGFlcGFpZW1rc3ByZXRnbXtydGNfYWVfZWZjfQ==",
    "aGFldTRlcGNhXzR0cmdte19yX2Ftbm1zZX0=",
]

for sample in samples:
    decoded = base64.b64decode(sample).decode()
    print(rail_fence_2_decode(decoded))
```

## 方法总结

- 核心技巧：检查前端游戏源码中的分数与奖励分支，对找到的字符串依次执行 Base64 和两栏栅栏解码。
- 识别信号：纯前端题给出一个格式正确却提交失败的结果，同时题干强调高分、金币或隐藏奖励时，应继续检查客户端常量和不可达分支。
- 复用要点：编码链相同不代表所有产物都是真 flag；应使用服务端提交结果、不同源码位置和题干提示共同判断诱饵与最终答案。
