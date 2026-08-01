# GlacierCTF2022 - Sandboxer

## 题目简述

服务先接收一个不超过 32 KiB 的 Base64 ELF，再要求登录。普通用户登录成功后，进程 `chroot` 到只含该 ELF 的临时目录、降低权限并执行它；`admin` 登录被显式禁用。目标是逆向自制密码哈希，借沙箱进程继承的文件描述符取得完整用户表，再恢复管理员密码。

## 解题过程

题面提示用户名 `steve`。第一次连接可随便上传占位数据，再用错误密码尝试登录；失败分支会打印存储的 Base64 哈希。逆向 `hash_avx` 后可知它并不是哈希，而是可逆变换：密码补零到 32 字节，与固定 32 字节 mask 异或，分成两个 AES-128-ECB 块加密，最后 Base64 编码。

反向解密 Steve 的条目：

```python
raw = base64.b64decode(stored_hash)
plain = AES.new(aes_key, AES.MODE_ECB).decrypt(raw)
password = bytes(a ^ b for a, b in zip(plain, xor_key)).rstrip(b"\0")
```

得到登录密码：

```text
H$g7FAKVR8f3&k!@wmMd6Vdk3rHSUrwg
```

第二次连接上传一个静态、自包含的小型 ELF，再用该密码登录。`read_userlist()` 在认证前打开 `./userlist.txt`，读取后既没有 `close(fd)`，也没有设置 `O_CLOEXEC`。因此后续的 `chroot`、`setuid` 和 `execv("payload", ...)` 都不会撤销这个已打开描述符。payload 枚举允许范围内的继承 fd，把能读取的数据写到标准输出，即可在空 chroot 内泄漏原目录中的完整用户表。

对 `admin` 的哈希执行同一 AES 解密与异或逆变换，恢复：

```text
glacierctf{Ju57_P0s1X_th1ng5}
```

## 方法总结

固定密钥 AES 不能把可逆密文伪装成密码哈希；密码应使用带随机 salt 的专用慢哈希。`chroot` 只改变后续路径解析，不会关闭此前打开的文件，降权也不会撤销已有 fd 的读取权。主线需要恢复自定义变换与程序资源生命周期，故归 Reverse。
