# find me

## 题目简述

题目是 OSINT + 流量解密。题面提示 WearyMeadow 在 Reddit 发帖，需要沿社交账号继续查找博客和 GitHub。GitHub 自动登录脚本泄露了复用密码，用该密码解开博客加密文章后，可得到通信加密算法；再从流量包中提取 key 并爆破随机种子，最终恢复 flag。

## 解题过程

题目描述里写到 WearyMeadow 通过 Reddit 发布了一篇帖子，这实际上是提示，先去 Reddit 搜索这个人来获取后续线索。

![Reddit 上 WearyMeadow 账号和发帖内容提供 OSINT 起点](WMCTF2023-find-me-wp/reddit-profile-clue.jpg)

先对内容做 base64 解码可获取附件地址：  
https://ufile.io/670unszp

这个临时外链本身只是附件下载地址，关键操作是对 Reddit 帖子内容做 base64 解码，得到后续待分析流量包；真正的解密线索不在下载站，而在该账号关联的博客和 GitHub。

当时还不清楚流量加密方式，无法直接解密。

注意该账号社交信息里有博客链接，头像也像 github 风格，这两个点都比较关键。

先看博客，里面只有一篇加密文章，暂时无法解密。

再看 github 页面，可确认其博客是 GitHub Pages，并且它有两个自动登录脚本。

这两个脚本泄露了账号密码，且两者一致：

```python
usernameStr = 'WearyMeadow'
passwordStr = 'P@sSW0rD123$%^'
```

这里存在密码复用风险，直接用该密码解锁被锁定的文章即可。

从该文章中可拿到通信服务的加密算法和初始密钥的确定方式：

```python
def encrypt(message, key):
    seed  = random.randint(0, 11451)
    random.seed(seed)
    encrypted = b''
    for i in range(len(message)):
        encrypted += bytes([message[i] ^ random.randint(0, 255)])
    cipher = AES.new(key, AES.MODE_ECB)
    encrypted = cipher.encrypt(pad(encrypted))
    return encrypted
```

再看流量包，找到 `SUCCESS` 条目可提取到 key：

```text
mysecretkey
```

加密方式较简单，只需写脚本爆破种子即可。

```python
import random
from Crypto.Cipher import AES
import string

table = string.printable
text = bytes.fromhex('xxx')

def pad(s):
    return s + b"\0" * (AES.block_size - len(s) % AES.block_size)

def is_printable(str_bytes):
    printable_count = 0
    total_count = len(str_bytes)
    
    for byte in str_bytes:
        if byte >= 32 and byte <= 126:
            printable_count += 1
    
    return printable_count / total_count >= 0.8

def decrypt(ciphertext, key):
    cipher = AES.new(key, AES.MODE_ECB)
    decrypted = cipher.decrypt(ciphertext)
    res = b''
    for i in range(11452):
        res = b''
        random.seed(i)
        for j in range(len(decrypted)):
            res += bytes([decrypted[j] ^ random.randint(0, 255)])
        if is_printable(res):
            print(res)

key = pad(b'mysecretkey')
decrypt(text, key)
```

提取流量中的所有数据并爆破后可拿到 flag。

```text
WMCTF{OH_Y0u_f1nd_Me__(@_@)}
```

## 方法总结

- 核心技巧：把 OSINT 线索链和流量加密还原结合起来，利用密码复用解锁博客文章，再恢复通信加密。
- 识别信号：题面给出用户名和社交平台时，应检查同名 GitHub、博客、头像来源、自动化脚本和复用凭据。
- 复用要点：临时附件外链要在 WP 中说明其承载的只是流量包；关键算法、key 来源和 seed 爆破范围必须写进正文。
