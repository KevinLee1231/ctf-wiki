# gesture

## 题目简述

附件是一个 JavaFX 九宫格解锁程序。用户依次经过九个节点后，程序把节点访问顺序映射为字符串，再执行 PBKDF2-HMAC-SHA1 校验；通过校验的字符串还会派生另一把 AES-CBC 密钥，用于解密内置 flag。

节点下标与字符的对应表直接保存在 `VerificationUtils.p_array`：

```text
节点：0 1 2 3 4 5 6 7 8
字符：g o 0 d $ l u c k
```

校验参数为 salt `deadbeef`、1500 轮、128 位输出，目标摘要是 `8045a9b6d9eece98352e353c9091f353`。

## 解题过程

`Grid` 保存的是“每个节点在第几步被访问”，`check()` 先把它反转成“第几步访问了哪个节点”，再按 `p_array` 生成密码。因此密码就是九个不同字符的一个排列，候选数只有：

$$
9! = 362880
$$

找到校验密码后，用 salt `11223344` 和相同的 PBKDF2 参数派生 16 字节 AES 密钥，再用源码中的 IV 对内置密文执行 AES-CBC-PKCS#7 解密。完整脚本如下：

```python
#!/usr/bin/env python3
from base64 import b64decode
from hashlib import pbkdf2_hmac
from itertools import permutations

from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad


characters = "go0d$luck"
check_salt = bytes.fromhex("deadbeef")
target = bytes.fromhex("8045a9b6d9eece98352e353c9091f353")

password = None
for candidate in permutations(characters):
    raw = "".join(candidate).encode()
    if pbkdf2_hmac("sha1", raw, check_salt, 1500, 16) == target:
        password = raw
        break

if password is None:
    raise RuntimeError("没有找到手势排列")

key = pbkdf2_hmac(
    "sha1",
    password,
    bytes.fromhex("11223344"),
    1500,
    16,
)
iv = b64decode("pJoKGZlx+tbr38ooZGNYeg==")
ciphertext = b64decode(
    "ajVD6Q7SS9ma7ghrOEG1Z1Tn0+RBlK/Rhntt4QVI8Iq0K6HZxkEfvVpnFk9utep2"
)
plaintext = AES.new(key, AES.MODE_CBC, iv).decrypt(ciphertext)

print("password:", password.decode())
print("flag:", unpad(plaintext, 16).decode())
```

唯一命中的密码为：

```text
$cuo0dklg
```

把每个字符反查回节点下标，得到九宫格路径：

```text
0 1 2
3 4 5
6 7 8

4 -> 7 -> 6 -> 1 -> 2 -> 3 -> 8 -> 5 -> 0
```

派生出的 AES 密钥为 `67e385c70660d3292497412436a0f2f9`，最终明文是：

```text
shellmates{that_false_sense_0f_secur1ty}
```

## 方法总结

本题虽然使用 PBKDF2 和 AES，但决定性步骤是从 Java 程序恢复“手势状态如何变成密码”的逻辑，因此归入 Reverse。PBKDF2 参数本身没有密码学漏洞；弱点是候选空间被程序约束成九个已知且互不重复字符的排列，穷举规模只有 $9!$。

分析 GUI 或 managed runtime 程序时，不必复刻整个界面。应沿事件处理函数追踪用户状态，定位最终校验与解密函数，再把状态转换边界提取成最小脚本。这里尤其要区分“节点到访问序号”和“访问序号到节点”两种数组方向，否则会得到手势的逆置换。
