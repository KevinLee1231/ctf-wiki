# find me

## 题目简述

题目是浏览器凭据取证。给定文件末尾藏有被破坏头尾的 ZIP，需要先根据 JPEG 文件尾和 ZIP 魔数修复压缩包；图片注释中的作者字段 `LSCHZNMHW` 是压缩包密码。压缩包内的 `Login Data` 是 Chrome 保存密码的 SQLite 数据库，需要导出 `password_value` 密文，再从 `lsass.dmp` 中提取 DPAPI Master Key，用 Mimikatz 解密 Chrome 密码 blob 得到 flag。

## 解题过程

用十六编辑器打开文件，拉到最后可以看到里面隐藏了一个压缩包。  
但是用 `binwalk` 无法识别，可以判断出文件头和文件尾被改掉了。  
先找到 `jpg` 文件的文件尾为 `FF D9` (00028592)，更改后面的 `0000` 为压缩包的文件头 `50 4B 03 04` 。  
然后拖到最后找到 `D5 01` 处后的 `00 00 00 00` 改成 `PK` ： `50 4B 05 06`  
恢复提取出来得到一个新的压缩包，有密码。  
在图片的注释信息里看到作者是 `LSCHZNMHW` 。得到压缩包密码： `LSCHZNMHW` 。  
里面有个 `Login Data` 。看文件头可以看出是 `SQLite` 数据库。  
这个 `Login Data` ，了解过 `Chrome` 浏览器的小伙伴应该知道， `Chrome` 保存的本地密码保存在这里。  
先导出密文：

```python
import sqlite3
import binascii
conn = sqlite3.connect("Login Data")
cursor = conn.cursor()
cursor.execute('SELECT action_url, username_value, password_value FROM logins')
for result in cursor.fetchall():
    print (binascii.b2a_hex(result[2]))
    f = open('test.txt', 'wb')
    f.write(result[2])
    f.close()
```

然后从 `lsass` 进程中提取出 `Master Key` ：

```bash
> sekurlsa::minidump lsass.dmp
> sekurlsa::dpapi
```

导出 `Master Key` 后系统会自动加入缓存，然后解密：

```bash
> dpapi::blob /in:test.txt
data : 64 33 63 74 66 7b 49 5f 4c 6f 56 65 5f 46 69 52 65 46 6f 78 21 40 23 7d # 密码的十六进制
```

十六进制转字符串就是最后的 `flag` ： `d3ctf{I_LoVe_FiReFox!@#}`

## 方法总结

- 核心技巧：先修复嵌入 ZIP，再按 Chrome `Login Data` + Windows DPAPI 的路径恢复保存密码。
- 识别信号：图片尾部有压缩包残留、压缩包内出现 `Login Data` 和 `lsass.dmp` 时，应联想到 Chrome 本地密码解密。
- 复用要点：Chrome 旧版密码密文保存在 SQLite 的 `logins.password_value`，DPAPI 解密需要对应用户的 Master Key；用 Mimikatz 时先加载 minidump，再让 Master Key 进入缓存后解 blob。

