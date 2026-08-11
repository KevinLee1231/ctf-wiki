# Go_Master

## 题目简述

附件是保留了大量符号信息的 Go ELF。程序先验证一个 9 字节字符串的 SHA-1，随后用该字符串拼出本地监听地址；真正的 flag 校验不从标准输入读取，而是在 `localhost:2333` 接收候选值，对其做 DES 加密和零填充后与内置密文比较。

## 解题过程

主函数首先用 `fmt.Fscanln` 读入字符串并检查长度为 9，随后计算 SHA-1。程序保存的摘要字节为：

```text
33 43 89 04 8b 87 2a 53 30 02 b3 4d 73 f8 c2 9f d0 9e fc 50
```

它对应：

```text
localhost
```

继续跟踪 `runtime.concatstring3` 和 `net.Listen`，可见该字符串会与 `:2333` 拼接。因此第一终端运行程序并输入 `localhost` 后，第二终端连接：

```bash
nc 127.0.0.1 2333
```

连接建立后会出现 `Give me KEY`。Go 的网络接收路径最终进入 `main.handleRequest`，把收到的 `[]byte` 转成字符串并调用 `main.Encrypt`。在该函数内可以直接识别：

```text
crypto/des.NewCipher
main.ZeroPadding
```

因此不必爆破候选 flag。逆向步骤是：

1. 沿 `crypto/des.NewCipher` 的实参回溯，取出样本实际传入的 8 字节 DES 密钥；不要把监听地址或界面提示误当成密钥。
2. 在最终 `runtime.memequal` 前定位内置密文，官方样本中位于 `unk_55790F` 一带，并按比较长度完整导出。
3. 按程序使用的分组方式解密。若 `main.Encrypt` 对每个 8 字节块直接调用 `Block.Encrypt`，对应 DES-ECB；若还有前一块异或，则按实际控制流还原 CBC。
4. 去掉 `main.ZeroPadding` 增加的尾部 `\x00`，得到应发送到 2333 端口的 flag。

对应的解密骨架如下，`key` 与 `ciphertext` 必须从题目样本填写：

```python
from Crypto.Cipher import DES

key = bytes.fromhex("...")          # des.NewCipher 的 8 字节实参
ciphertext = bytes.fromhex("...")   # unk_55790F 处完整常量

plain = DES.new(key, DES.MODE_ECB).decrypt(ciphertext)
print(plain.rstrip(b"\x00"))
```

原 PDF 没有以文本给出密钥、密文或最终 flag，截图也只展示了常量地址，因此本文不伪造这些值。公开复盘确认了相同的 `localhost:2333 -> handleRequest -> DES -> 比较` 路径，读者即使不打开外链，也可以按上面的样本取值步骤完整复现；补充来源为 [HGAME2020 RE/PWN 复盘](https://blog.51cto.com/u_14601424/6286037)。

## 方法总结

- 核心技巧：利用 Go 符号名快速还原调用链，并识别标准库 SHA-1、网络监听与 DES API。
- 关键细节：第一阶段输入只决定监听地址；真正的候选值来自本地 TCP 连接。DES 密钥必须以 `des.NewCipher` 的实际 8 字节实参为准。
- 证据边界：原始 WP 足以闭合算法和复现路径，但没有提供样本常量的文本值，所以不能负责任地补写最终 flag。
