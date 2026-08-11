# まひと

## 题目简述

题目把多种古典编码和密码首尾相接：先把 flag 做 ROT13，再整体逆序，以宽度 6 做栅栏换位，使用密钥 `Liki` 做 Vigenère 加密，随后依次进行 Base64、ASCII 十进制和摩斯编码。解题时按完全相反的顺序逐层剥离即可。

## 解题过程

原始摩斯串使用 `/` 分隔字符，其中不仅有数字的摩斯编码，还包含代表斜杠的 `-..-.`。先解出十进制 ASCII 序列，再把每个数转成字符，可以得到以下 Base64 文本：

```text
VmlnZW5lcmUtTGlraTp9VmttdkpiITFYdEF4ZSFocE0xe00rOXhxenJUTV9Oan5jUmc0eA==
```

Base64 解码结果为：

```text
Vigenere-Liki:}VkmvJb!1XtAxe!hpM1{M+9xqzrTM_Nj~cRg4x
```

冒号前给出了下一层算法和密钥。Vigenère 解密只在遇到英文字母时推进密钥，标点和数字原样保留，得到：

```text
}KccnYt!1NlPpu!zeE1{C+9pfrhLB_Fz~uGy4n
```

这里的“栅栏 6”不是常见的折返式 Rail Fence，而是把明文按下标模 6 分组后依次拼接，即加密操作为 `plain[0::6] + plain[1::6] + ... + plain[5::6]`。撤销换位后得到：

```text
}!!Ch~K1z+LucNe9BGclEp_ynP1fF4Yp{rzntu
```

将整串逆序，再对字母执行 ROT13：

```python
import codecs

stage = "}!!Ch~K1z+LucNe9BGclEp_ynP1fF4Yp{rzntu"
print(codecs.decode(stage[::-1], "rot_13"))
```

最终得到：

```text
hgame{cL4Ss1Cal_cRypTO9rAphY+m1X~uP!!}
```

## 方法总结

多层编码题最重要的是记录每层的输入表示、输出表示和方向，不能只看算法名称连续尝试。格式也是有效约束：Vigenère 解密后首字符仍是 `}`，说明它应当通过后续逆序回到 flag 末尾；这一点可以用来确定栅栏换位的还原方向以及 ROT13 所处的位置。
