# Mazeware

## 题目简述

程序表面是三层迷宫，实际在运行时解密两段 shellcode、寻找 libc 代码洞并劫持 `getchar@GOT`，由隐藏钩子检查移动轨迹。仓库同时保留了生成端数据，因此不必手工走完迷宫：从二进制中恢复 32 字节 RC4 密钥与真实密文即可直接解密 flag。

## 解题过程

初始化代码先根据 libc 映射寻找可执行代码洞，把由固定 32 字节密钥异或保护的 shellcode 解密进去，再将 `getchar@GOT` 改为代码洞中的入口。每次读取方向键都会进入钩子；走错路线会破坏下一阶段数据，走对三层才会到达输出函数。这解释了为何只分析主循环会看到普通迷宫，却无法找到完整判定逻辑。

决定性信息仍以静态数据形式留在 ELF 中：

```text
key = 44 55 62 1d 5d 46 f9 2c 32 5e 62 5f b5 95 f6 9e
      67 4b 3a 29 98 0c 12 90 19 e8 c1 b4 f7 a6 0b 22
```

生成端 `enc_flag.py` 使用标准 RC4，并明确区分真实 flag 与作为诱饵的 YouTube URL。`enc_fake_flag` 是诱饵的 RC4 密文，`enc_flag_xor` 则把两份密文和循环常量混在一起；撤销这层关系后才能得到真实密文。用二进制中的数据复现如下：

```python
from Crypto.Cipher import ARC4

key = bytes.fromhex(
    "4455621d5d46f92c325e625fb595f69e"
    "674b3a29980c129019e8c1b4f7a60b22"
)

fake_ct = bytes.fromhex(
    "8bf24a8b9eeb29e637f0b3f4b9be1f1753e32f4b4e6706cf"
    "06ca84e1bb0b383ec58cc9a8721d3cbbbe26b8"
)
mixed = bytes.fromhex(
    "2283bd54afe519943ec1b52cefb62ddb34d3f81df1e90d98"
    "6dbbeb0ba9ee12802cb0be58e5bc3181418380"
)
mask = (0xdf07b7a75dac852d).to_bytes(8, "little")
real_ct = bytes(fake_ct[i] ^ mixed[i] ^ mask[i % 8]
                for i in range(len(fake_ct)))
print(ARC4.new(key).decrypt(real_ct).decode())
```

输出的真实明文为：

```text
grey{h1dd3n_1n_pl41n51gh7_35ffcbede152a94e}
```

这里不能调用表面上的 `win()` 就草率结束：源码中该函数故意解密 `enc_fake_flag`，输出的是诱饵 URL；真实 flag 对应另一组密文关系。

## 方法总结

本题把控制流藏进 GOT 钩子和运行时 shellcode，但加密材料仍必须存在于进程内。先识别代码洞、`getchar` 劫持和分阶段校验的用途，再沿数据流区分真实密文与诱饵，能避免在复杂迷宫状态机上浪费时间。静态解密也应以二进制中的数组为准，生成脚本仅用于交叉验证。
