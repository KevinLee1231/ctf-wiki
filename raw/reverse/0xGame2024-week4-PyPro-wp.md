# PyPro

## 题目简述

题目附件 `PyPro.exe` 是由 PyInstaller 打包的 Python 程序。解包后可确认字节码版本为 Python 3.12；校验逻辑会把 44 字节输入补齐到 48 字节，使用固定密钥进行 AES-ECB 加密，再将结果 Base64 编码后与常量比较。

## 解题过程

先用 `pyinstxtractor.py` 解包可执行文件。反编译 Python 3.12 的 `.pyc` 时，旧版 `uncompyle6` 不适用，可以使用 [pycdc 官方源码仓库](https://github.com/zrax/pycdc) 自行编译 `pycdc`，或者用 `pycdas` 阅读字节码。这里保留该链接，是因为它是所用反编译工具的上游项目；本题所需的关键逻辑已完整整理如下，无需依赖外部教程：

```python
import base64
from Crypto.Cipher import AES
from Crypto.Util.number import long_to_bytes

key = 0x554B134A029DE539438BD18604BF114

def pad_to_48(data):
    length = 48 - len(data)
    return data.ljust(48, length.to_bytes())

flag = input().encode()
assert len(flag) == 44
cipher = AES.new(long_to_bytes(key), AES.MODE_ECB)
result = base64.b64encode(cipher.encrypt(pad_to_48(flag)))
assert result == (
    b"2e8Ugcv8lKVhL3gkv3grJGNE3UqkjlvKqCgJSGRNHHEk98Kd0wv6s60GpAUsU+8Q"
)
```

`long_to_bytes(key)` 得到的实际 AES 密钥为：

```text
0554b134a029de539438bd18604bf114
```

原函数名虽然写成 `PKCS5_pad`，但对于长度为 44 的输入，它实际追加四个 `0x04`，恰好是合法的 PKCS#7 填充。因此直接逆序执行 Base64 解码、AES-ECB 解密和去填充即可：

```python
import base64
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = bytes.fromhex("0554b134a029de539438bd18604bf114")
target = base64.b64decode(
    "2e8Ugcv8lKVhL3gkv3grJGNE3UqkjlvKqCgJSGRNHHEk98Kd0wv6s60GpAUsU+8Q"
)

plain = AES.new(key, AES.MODE_ECB).decrypt(target)
print(unpad(plain, AES.block_size).decode())
```

输出为：

```text
0xGame{1cb76d38-4900-476f-bf1b-9d59f74d7b2e}
```

## 方法总结

这类 PyInstaller 逆向题的重点是先确认打包方式和 Python 字节码版本，再恢复最终比较链。即使反编译器不能完整生成源码，只要从字节码中取得密钥、工作模式、填充规则和目标密文，也足以按相反顺序还原输入。本题尤其要注意整数密钥转换后带有一个前导零字节，以及自定义函数实际上产生的是四字节 `0x04` 填充。
