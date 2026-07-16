# 绯想天则

## 题目简述

题目是一个使用 C# 编写的 Unity 小游戏。该程序采用 Mono 后端而非 IL2CPP，核心托管逻辑保存在 `Assembly-CSharp.dll` 中，可以直接使用 dnSpy、ILSpy 等 .NET 反编译器分析。击败 Boss 后，`EnemyStats.WinFlag()` 会解密预置密文并把明文写入界面，因此既可以静态解开 RSA 密文，也可以修改游戏逻辑触发正常胜利流程。

## 解题过程

### 定位 flag 解密入口

在 `Assembly-CSharp.dll` 中搜索与敌人状态和胜利相关的方法，可以定位到 `EnemyStats.WinFlag()`：

```csharp
private void WinFlag()
{
    this.EncryptflagText.text = this.Decrypt(
        this.privateKeyXML,
        this.EncryptflagText.text
    );
}
```

同一类中还保存了 Base64 编码的 `EncryptText` 和 XML 格式的完整 RSA 私钥 `privateKeyXML`。密文解码后为 256 字节，对应一组 2048 位 RSA 密文；解密使用 PKCS#1 v1.5 填充。

### 使用内置私钥静态解密

.NET 的 `RSAKeyValue` XML 分别给出 `Modulus`、`Exponent`、`D`、`P`、`Q` 等大整数，各字段均为大端整数的 Base64 表示。解析其中的 $n$、$e$、$d$、$p$、$q$ 后即可重建私钥并解密：

```python
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
import base64
import xml.etree.ElementTree as ET

EncryptText = ("TxDTykBFS+ewdHSIKSRRw63i4/7Os3pHqvC/WOnnFPKupfz0VpUqFRVEgtOdfJLdcxNKBvc67E/nIqU4dz8L88DZB9xVaKawdDgJ8c3EhI//r1aZkzbN2hURFdsS1UoF932XT9wPW61TsXPX7UyFAG4IInifxQr6KU7hyQNHIkpBkYsTgp/wxH5/g6IKEff8H/zEjbLcalck5k/r3vG8DeBQkhZUQ4I5HfIyetUBAmZyr8sZkygErKxVaF6vWZZtoPBkLzCxY0WIoaYcTfp+n5q1dUBDOshvXhVf1M3KGokZL2PqXT9K9Gv91d8aazXq7MV1iqCN4AwWAeFPn8OIYg=="
)
privateKeyXML = """<RSAKeyValue><Modulus>jR+j5ZyLvbOyQRZgJQtHl5slRSQ4igoTQrKrwnmIwOcrkc48RQ6qD6wFXYAbFmiu34YpO638surG2dFemO0+vSoJuFksK70amen3pdilSgUukdFZ3SEJ/7QoeWls40exLsVPn3zWclN5b1ESaL4TN0MpC9lJUuWqkwiYSU2fZYQg9Z7S05RcVtl0JgK9O/DmLjW6+t6RdUAyo+9gsPXdGNOd/f4LyOTbj0iaWg2aCGfZhz3UtqACS2q6L9p1a9L3vYgLNpp1GnUcZnA+CVfhzcCWTHQ4X7hTuJsyysFEm/6tei3uBKYq6/HnTrmo2rSZxKS/GuExRO3A9hmyD+PEcw==</Modulus><Exponent>AQAB</Exponent><P>+6YUEMyFZlINlnZYRznRU0qU2Krb4QaFCUnNhFsL37y55rnz0/hLfLSA2FistQiMLy0Yz01fkxOTtwwkWkHikRu7KFi7vQzTg/KbdKNnHJ11huemk0DuwxDghj+3uNC30heSA6NFlF47zlge8AY2J+iQ8JpgJNsrfZYNRZ5txD0=</P>
<Q>j5BSrK3Li9VaBvRn2UGMf8q2N0J7tbaRvQonHhU1uBQYlhifttmK5wr1PurZHsb7tiNIv0RluM9HaUbQ/CczqLskVKJNkwpVyGM2rydiqbZ/KaP00ytTiR7Hxd0GUC06Syfi2h1VwaEewAOTkRUo/DuKYugPSaiyqARipyhfRm8=</Q><DP>0lwCagiNevscYKqNIP0z/mxaAMTTCUhp7VnEct+pDV62CClpqcflUlmRW0jFFpAOn2ETXDdRraCv2lRMDycEPkjwKsoCJgaSyboEOXxetYzqsdrzZCTjciypg4/ABL506yrI5EGX6G7dj6AaPIr0umeuwXJK7IRJ1rGYZpoJKAE=</DP><DQ>HA4BCfughj/4KtnCHYOguCxd9WiJklYOHtoIEOnmKIXM1DAVrf7PFR1gFZ6BNXF/KPW2NqJgGoBvHRSYrF3gy31euSdKb4yafOFeg1X4AuBF81Y19rpFxcr9ER6DKFHeTWeK/kKzSnZ48t8ADF8NNlVQUsm0ixlraEgLG01ZaQM=</DQ><InverseQ>qZwSExM5GW+CsTtvN2DPmwQtLf7cTA2DwY+8+4bnR7X0MTqssUDNtc8l+HD3uml9BRg9m4t1PoY+qGsh5qEtFsITBvt0pqNM5mdg8WrB1e85Xzpdvsfqvv1PN3KsDjduzkY0SazVBlur+OX86JTF3jSLFeFkornO1ckHcApqU54=</InverseQ><D>LABbh/IhmAp5X9XsMGCt99VF76L1hgTSMI+pAkAGpa7uZM3a+OUznSNToO2ahIgrTkJ0hMkg62BMlAm15xTB5RVAZpxXK2QQ8UCEGM/N6aBn/ss5q7rrdTDlFcYLT2pBEoYu51lzO75PNKggh0wMjcSA/dLIC/LUFngtk12Cf5IRwsdn0Kc2aoiiHU0NtvWklLZGkzcs4VL5FI7n1yDeIS9I+lKA14dJUCEhAltfMMiTSq+vpzoSedOcc8V0v9O9mjef8W62DasLusKxCCQi6PzsLA7ul2yLuOWTi1oysnrUVZryOWTvMr2V3GC0wfGRdXajd6qwwKVvGUt72egeEQ==</D></RSAKeyValue>"""
def b64_to_int(s):
    return int.from_bytes(base64.b64decode(s), byteorder='big')
root = ET.fromstring(privateKeyXML)
fields = {child.tag: child.text.strip() for child in root}
n = b64_to_int(fields['Modulus'])
e = b64_to_int(fields['Exponent'])
d = b64_to_int(fields['D'])
p = b64_to_int(fields['P'])
q = b64_to_int(fields['Q'])
rsa_key = RSA.construct((n, e, d, p, q))
cipher = PKCS1_v1_5.new(rsa_key)
plaintext = cipher.decrypt(base64.b64decode(EncryptText), b'')
print(plaintext.decode('utf-8'))
```

运行结果为：

```text
0xGame{TenshiSamaSaiko_IWANNATENSHIFUMO_INESBATTLE}
```

### 修改血量触发胜利流程

游戏也提供了动态解法：击败 Boss 后会调用上述 `WinFlag()`。正常情况下玩家攻击力只有 10，而 Boss 血量约为 50000，直接通关成本很高。`CharacterStats.GetMaxHealthValue()` 的原逻辑为：

```csharp
public int GetMaxHealthValue()
{
    return this.maxHealth.GetValue() + this.vitality.GetValue() * 5;
}
```

可在 dnSpy 中把返回值改为 1，保存修改后的程序集，再进入游戏攻击 Boss。需要注意，这个方法属于通用的 `CharacterStats`，直接修改会同时影响玩家和 Boss；双方都可能被一击击败，因此应先命中 Boss。若只追求稳定取 flag，前面的静态 RSA 解密更直接。

## 方法总结

Mono Unity 题首先应检查 `Assembly-CSharp.dll`，从触发 flag 的业务方法反向追踪数据。本题把密文和完整私钥同时放在 `EnemyStats` 中，静态重建 RSA 私钥即可得到答案；修改血量则验证了游戏内的正常触发链。相比只展示反编译器截图，正文记录类名、方法名、算法、密钥格式和可执行脚本，才能让解法脱离特定 GUI 后仍可复现。
