# Pickle Rick

## 题目简述

附件不是可直接执行的程序，而是一串由单词 `pickle` 和 `rick` 组成的数据。题目名提示这两个单词代表二进制状态；还原后仍有一层逐字节变换。

## 解题过程

先把 `pickle` 记为 1、`rick` 记为 0，按原顺序每八位组成一个字节。直接观察所得字节还不是正常文件，但对每个字节异或 `0x67` 后，开头恢复为 ELF 魔数 `7f 45 4c 46`，说明映射和位序正确。

核心还原逻辑可以写成：

    words = open("message.txt", encoding="utf-8").read().split()
    bits = "".join("1" if word == "pickle" else "0" for word in words)
    data = bytes(int(bits[i:i + 8], 2) ^ 0x67 for i in range(0, len(bits), 8))
    open("recovered", "wb").write(data)

为恢复文件添加执行权限并运行，或直接检查其中的字符串，即可得到：

    byuctf{1m_p1ckl3_r1111ck!}

## 方法总结

题目的两层编码分别是词语二值化和固定字节异或。还原未知二进制时，应利用 ELF、PNG、ZIP 等文件魔数验证映射、位序与密钥，而不是只以“能解出可打印文本”为判断标准。
