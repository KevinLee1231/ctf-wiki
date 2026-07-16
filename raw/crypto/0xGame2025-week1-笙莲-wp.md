# 笙莲

## 题目简述

题目先以 GB2312 将含中文的 flag 编码为字节，再等分成四段，分别进行 Base64、十六进制、自定义三进制表示和“小端整数七次幂”变换。解题时必须逐段使用对应逆操作，并在拼接全部字节后统一以 GB2312 解码。

## 解题过程

### 编码与字节序

flag 被 encode 后拆成了四部分，用四种方法分别简单加密，解密时分块解密，拼一起后再 decode。

编码（encode）/解码（decode）：计算机中信息是以字节（Byte）的方式存储的，$1\text{Byte}=8\text{Bit}$，每个字节的取值范围为 0-255。文字信息在储存、传输时，必须先将它们映射至字节序列，此即“编码”；读取时亦须将之从字节序列转换回文字，此即“解码”。

编码有很多种，最常用的（也是 Python 默认的）是 UTF-8，其能表达世界上绝大多数的文字符号。UTF-8（及大多数编码）下，ASCII 字符（0-127）可以用单个字节表示；其余字符可以通过多字节序列表示。

如“你好”的 UTF-8 编码结果是 `b'\xe4\xbd\xa0\xe5\xa5\xbd'`，其中 `b` 是 Python 中表示字节字面量的前缀，`\x??` 表示这是一个十六进制下值为 `??` 的字节字面量，当这个字节本身不能映射到可打印的 ASCII 字符时就会如此表示。

UTF-8 中常见汉字通常编码为三个字节。题目使用 `gb2312`，其中大多数汉字占两个字节，例如“你好”的编码结果是 `b'\xc4\xe3\xba\xc3'`。

对于含有非 ASCII 字符的文本，编码与解码使用不同码表时会得到错误字符或直接抛出异常。二进制文件本身也不一定是文本，不能仅凭“出现乱码”判断其编码。

例如，把 GB2312 字节直接按 UTF-8 解码会立刻失败；加入 `errors="ignore"` 只会丢弃无法解释的字节，并不能恢复原文：

```python
text = "欢迎来到0xGame2025，祝各位在Crypto的天空中展翅翱翔awa"
raw = text.encode("gb2312")
raw.decode("utf-8")
# UnicodeDecodeError: 'utf-8' codec can't decode byte 0xbb ...
```

### 四段变换

```python
if __name__=='__main__':
    flags = [flag[i*len(flag)//4:(i+1)*len(flag)//4] for i in range(4)]
    ciphertexts = []

    c0 = b64encode(flags[0])
    c1 = flags[1].hex()
    c2 = awaqaq(flags[2])
    c3 = int.from_bytes(flags[3],'little') ** 7

    print(c0)
    print(c1)
    print(c2)
    print(c3)
```

`c0` 采用 Base64，将任意字节表示为便于传输的可打印字符；逆操作是 `b64decode`。

`c1` 使用 `bytes.hex()` 输出十六进制字符串，逆操作是 `bytes.fromhex()`。

`c2` 把字节对应的大整数写成自定义三进制，其中 `a/w/q` 分别代表数字 `0/1/2`。按位还原三进制整数后再转回字节即可。

`c3` 先按小端序把字节转成整数，再计算七次幂。密文是精确整数幂，因此使用 `gmpy2.iroot(c3, 7)` 取整数七次根，不需要模运算。

字节和整数之间的转化及“小端序”/“大端序”（little/big-endian）：

将字节序列转化为整数非常简单，将之以 0-f 的字符写出后按十六进制转化即可。Python 内置的函数 `int.from_bytes()` 和 `int.to_bytes()` 提供了整数从/向字节的原生转换。

默认的大端序把高位字节放在前面，例如 `\x12\x34` 对应 `0x1234`；小端序把低位字节放在前面，同一字节序列对应 `0x3412`。

**注意小端序并不是将整个 `.hex()` 后的字符串反转，而是将字节间的排列顺序反转，单个字节内的内容不变，如上。**

Python 的 `int.from_bytes()` 与 `int.to_bytes()` 都可以显式传入 `byteorder='little'`，还原 `c3` 时必须与加密端保持一致。

### 完整求解脚本

```python
from Crypto.Util.number import *
from base64 import b64decode
from gmpy2 import iroot
from functools import reduce

def inv_awaqaq(s:str):
    return reduce(lambda x,y: x*3+y,map(lambda x : {'a':0,'w':1,'q':2}[x],reversed(s)),0).to_bytes(25)

if __name__=='__main__':
    c0 = b'MHhHYW1le7u2063AtLW9MHhHYW1lMjAyNQ=='
    c1 = 'a3accfd6d4dac4e3d2d1beadd1a7bbe143727970746fb5c4bb'
    c2 = 'wqwwwqqaawwwaaqawqwawwwwaaawwwawaqqwwwqaqwwqwaaqwaqqaaawqqqaqaqwaaawwwqaqaaaaqawaqqqwwqqwaqwqwwwawawqqwwqqawqwaqwwawwqwaqqaqwaw'
    c3 = 5787980659359196741038715872684190805073807486263453249083702093905274294594502252203577660251756609738877887210677202141957646934092054500618364441642896304387589669635034683021946777034215355675802286923927161922717560413551789421376288823912349463080999424773600185557948875343480056576969695671340947861706467351885610345887785319870159654836532664189086047061137903149197973327299859185905186913896041309284477616128

    msgs = [b'' for _ in range(4)]
    msgs[0] = b64decode(c0)
    msgs[1] = bytes.fromhex(c1)
    msgs[2] = inv_awaqaq(c2)
    msgs[3] = int(iroot(c3,7)[0]).to_bytes(25,'little')

    print(b''.join(msgs).decode('gb2312',errors='ignore'))
```

脚本分别恢复四段原始字节，按原顺序拼接后才执行 GB2312 解码。若提前逐段解码，分段边界可能切在多字节汉字中间，造成错误。

## 方法总结

- 核心技巧：识别四种独立可逆变换，逐段恢复字节并按原编码统一解码。
- 识别信号：Base64 字节串、十六进制文本、自定义有限字符码表以及无模数的大整数幂。
- 复用要点：区分“文本编码”和“表示层编码”，同时严格保持字节序；多字节字符应在完整字节流重组后再解码。
