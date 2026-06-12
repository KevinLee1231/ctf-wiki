# Kilogram

## 题目简述

题目是 VMP 保护过的文件加密器逆向。实际算法是：读取 `flag`，生成随机本地密钥 `_localKey`，再用固定 passcode `Lilac+present` 派生出的 `_passcodeKey` 加密 `_localKey`，最后把若干字段写入 `flag.enc`。

输出格式由源码可知：

```cpp
header = "lilac___"
output = header || encrypted_localKey || passcodeKeySalt || encrypted_flag || md5(encrypted_flag)
```

## 解题过程

### 文件结构

`main` 中先随机生成 `pass` 和 `salt`，派生 `_localKey`：

```cpp
vector<uint8_t> _localKey = CreateLocalKey(pass, salt);
```

随后固定 passcode：

```cpp
passcodeKeyData = {'L', 'i', 'l', 'a', 'c', '+', 'p', 'r', 'e', 's', 'e', 'n', 't'};
_passcodeKey = CreateLocalKey(passcodeKeyData, _passcodeKeySalt);
```

`CreateLocalKey` 先计算 `SHA512(salt || passcode)`，再用 PBKDF2-HMAC 派生 64 字节 key。

### 解密主线

加密分两层：

1. 用 `_localKey` 初始化一个类 RC4 的 256 字节置换表，异或加密 flag 内容。
2. 用 `_passcodeKey` 初始化同样的置换表，异或加密 `_localKey`。

因为 `_passcodeKeySalt` 明文存放在 `flag.enc` 中，而 passcode 固定为 `Lilac+present`，可以先重新派生 `_passcodeKey`，解出 `_localKey`。再用 `_localKey` 对 `encrypted_flag` 做同样的异或流程，即可还原 flag。

校验字段是：

```cpp
md5_hash = md5(buffer);
```

这里的 `buffer` 是第一层加密后的 flag 密文，因此可用末尾 MD5 判断文件分割和第一层密文是否取对。

### VMP 处理

若只拿到二进制，需要先处理 VMP 壳。官方备注中提到旧版本 VMP 可以用现有 unpacker 辅助处理；不过 WP 中真正需要保留的是文件格式、密钥派生和两层流加密关系，不需要贴完整反汇编。

## 方法总结

- 核心技巧：从输出格式拆出 salt、加密本地密钥和密文，利用固定 passcode 先解本地密钥，再解 flag。
- 识别信号：加密文件中同时保存 passcode salt 和被 passcode key 保护的 file key，这是典型“口令保护本地随机 key”的结构。
- 复用要点：遇到 VMP/壳题时先判断是否有源码或可恢复的高层算法；真正影响解题的是密钥派生和文件格式，不是哈希/加密函数的大段实现。
