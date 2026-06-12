# AliceWantFlag

## 题目简述

本题是交互式 ElGamal/认证协议题。服务端有注册、登录功能，目标是以 Alice 身份发送加密信息拿 flag；攻击者可以让 Alice 连接自己伪造的服务器，从而得到 Alice 发送的若干加密数据。

题目源码和 exp 仓库中包含 ElGamal 交互服务、服务器伪造脚本和 Boneh 相关参考论文。核心目标是恢复 `Alicepasswd` 和 `endkey`：预期利用 ElGamal 乘法同态逐位泄露密码，再对短 `endkey` 做中间相遇；非预期则利用 AES key 未补齐导致的长度报错，通过类似位泄露和二分恢复。

## 解题过程

题目源码、详细 exp 和参考文献仓库见 [`shal10w/d3ctf2021_AliceWantFlag`](https://github.com/shal10w/d3ctf2021_AliceWantFlag)。该仓库支撑的关键内容已经转写到下文：ElGamal 乘法同态、`signup` 长度 oracle、`endkey` 中间相遇和 AES key 长度非预期。

题目代码较长，核心是 server 提供注册、登录功能，以 Alice 账号登录后发送加密信息拿 flag。攻击者连接 Alice 后可以向她提供自己伪造的服务器地址，因此能得到 Alice 发出的部分密文信息。关键条件如下：

1. 与 Alice 交互时，可以给她 `r`，并得到服务器公钥加密的 `r ^ Alicepasswd`。
2. server 在 `signin` 和 `signup` 中各有一次解密；`signin` 只判断值是否正确，回显信息少，`signup` 解密后会按长度给出不同反馈，更适合作 oracle。
3. 非预期路径中，Alice 会解密 `endkey`，并把结果与 `r ^ Alicepasswd` 拼接成 AES key；AES key 没有做长度补全，因此长度错误也能形成 oracle。

题目目标为得到Alicepasswd与endkey

预期解
elgamal有乘法同态特性,即

若 ElGamal 密文写成 `E(m) = (y1, y2)`，则在不知道私钥的情况下把第二个分量改成 `k*y2`，解密结果会变成 `k*m`：

```text
D(y1, k*y2) = k*m
```

通过这个方式可以在不知道明文的情况下修改明文。比如乘以 2 能将明文左移一位，从而可能触发 `signup` 的长度判断。正常情况下这只能触发一次，得到最高位信息；因此需要利用 `r` 修改 Alice 发送的明文，把当前最高位异或为 0，再继续用长度反馈爆出下一位，最多约 88 次即可恢复密码。

`endkey` 很短，只有 5 字节，可以使用中间相遇。《Why Textbook ElGamal and RSA Encryption Are Insecure》中提到，一个 40 bit 以下的数有一定概率能分解为两个 20 bit 以下数的乘积。
若 `m = a*b`，则有 `pow(y2, q, p) = pow(m, q, p) = pow(a, q, p) * pow(b, q, p)`。因此可以枚举两侧 20 bit 以内的因子做中间相遇，恢复 `endkey`。中间相遇过程可以只截取一部分来减少空间和时间，大约几十秒即可完成一次。

非预期解(最后绝大多数队都是这么干的)
由上面第三点可知，AES key 没有进行填充，因此可以通过报错知道长度。先用与预期相同的方法解出 `passwd`；接着利用 AES key 长度必须为 16 的限制，对 5 字节 `endkey` 二分找到一个 `k`，使得：

```text
k * endkey < 2^128 < (k + 1) * endkey
```

于是 `endkey` 只能落在 `2^128 // k` 附近，检查相邻候选即可。

题目源码与 exp 的关键依赖是：服务端接受同态构造后的 ElGamal 密文并反馈比较结果，因此可以把“密码某一位是否满足条件”转成 oracle；参考论文提供 textbook ElGamal 同态与选择密文风险的背景。

## 方法总结

- 核心技巧：利用 ElGamal 乘法同态修改密文对应的明文位，再借注册/登录回显或 AES key 长度错误构造 oracle。
- 识别信号：攻击者能让 Alice 向自控服务器发送加密消息，且服务端对解密结果的长度/正确性有可区分回显。
- 复用要点：交互式密码题要把“可控服务器”“可控随机数 r”“错误回显”都视为 oracle；同态加密即使不能直接解密，也可能允许按位移动或乘法修改明文。

