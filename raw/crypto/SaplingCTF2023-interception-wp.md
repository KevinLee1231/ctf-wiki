# interception

## 题目简述

题目给出一张写有密钥的照片和若干十六进制密文。照片中的内容不是需要保留的视觉证据，只是一串可直接转写的 AES 密钥：

~~~text
1666b7e028d54f193895dbc3bddba88a
~~~

密文按 16 字节分组，结合明文长度与重复块可以判断使用的是 AES-ECB。目标是用照片中的 16 字节密钥逐条解密消息。

## 解题过程

先把十六进制密钥还原为字节，再用 ECB 模式解密每一行。明文采用 PKCS#7 填充，因此解密后要检查并去掉末尾填充：

~~~python
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

key = bytes.fromhex("1666b7e028d54f193895dbc3bddba88a")
cipher = AES.new(key, AES.MODE_ECB)

with open("messages.txt", "r", encoding="utf-8") as f:
    for line in f:
        ct = bytes.fromhex(line.strip())
        print(unpad(cipher.decrypt(ct), 16).decode())
~~~

逐条查看明文即可找到：

~~~text
maple{5upeR_5ECReT_ExCh4nGE}
~~~

## 方法总结

本题的重点是把照片中的文本可靠转写为 32 个十六进制字符，并正确区分字符而不是依赖 OCR 猜测。看到密文长度是 16 的倍数且存在重复分组时，应优先检查 ECB；解密后若末尾出现重复的低值字节，再按 PKCS#7 去填充。
