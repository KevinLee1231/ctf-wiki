# week4re2

## 题目简述

题目给出一个实现了分组加密的程序。程序内含固定密钥和 48 字节密文，需要识别算法、加密模式与填充方式，进而恢复明文。

## 解题过程

程序中可以识别出 AES 的 S 盒、轮常量以及密钥扩展逻辑；其后依次出现字节代换、行移位、列混合和轮密钥加等标准轮函数。因此算法是 AES-128，而不是自定义密码。

继续跟踪调用参数可知：

- 密钥固定为从 `0x00` 到 `0x0f` 的 16 个字节；
- 每个 16 字节分组独立加密，没有 IV，也没有前一分组参与运算，因此模式为 ECB；
- 最后一组明文以零字节补齐，而不是 PKCS#7 填充。

直接使用相同参数解密题目内置的三个分组：

~~~python
from Crypto.Cipher import AES

key = bytes(range(16))
ciphertext = bytes.fromhex(
    "1892f613761123eff70ab761461610c8"
    "4da194c18a2e8fba568810956edf74ee"
    "55812d305905ed63d27c1347a9a3f620"
)

cipher = AES.new(key, AES.MODE_ECB)
plaintext = cipher.decrypt(ciphertext).rstrip(b"\x00")
print(plaintext.decode())
~~~

输出为：

~~~text
flag{2f2d0017-80f6-4f6f-97c9-bc4e9b21f3b1}
~~~

## 方法总结

识别常见密码实现时，常量表和轮函数结构比函数名更可靠。确认 AES 后仍需继续核对密钥长度、分组间是否有关联、是否使用 IV，以及末块如何填充；这些参数任意一项判断错误，都会导致解密结果不正确。
