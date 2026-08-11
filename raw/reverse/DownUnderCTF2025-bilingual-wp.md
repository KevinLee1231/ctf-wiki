# bilingual

## 题目简述

手册是一个经过压缩/最小化的 Python 脚本：它解压内嵌 DLL 为 `hello.bin`，用 `ctypes` 调用 DLL 中的四个检查函数，并以通过检查的 password 作为 RC4 密钥解密 Base64 密文。验证逻辑故意横跨 Python 和 C++：DLL 又通过回调让 Python 对由 C++ 拼出的表达式执行 `eval()`，因此只阅读任意一侧都无法完整恢复条件。

目标 password 长度必须为 12。源码给出了最终 RC4 密文和解密函数，故关键工作是从 C++ 的状态机与 Python 回调还原所有字符，而不是猜测或暴力枚举 flag。

## 解题过程

### 前两个检查固定开头和中段

`Check1` 检查 `password[0] ^ 0x43 == 11`，因此首字符为 `H`，并更新 `g_State[0] = 'H' | 0x72 = 0x7a`。`Check2` 通过 Python 回调取得第 5、6 位并各加 3；两条算式恢复出 `p[5] = 'p'`、`p[6] = 'h'`，同时令 `g_State[1] = 0x6d`。

`Check3` 从局部常量经过异或构造字符串 `PASSWORD`，再让 Python 求值 `ord(PASSWORD[i])`。它约束第 7、8、11 位分别为 `1`、`1`、`3`，并利用表达式中的字符 `0` 固定第 4 位。这个跨语言回调的本质只是读取 Python 全局 `PASSWORD`；无需把 DLL 当作独立黑盒执行。

### 用状态解密表达式并恢复剩余字符

`Check4` 以目前的 `g_State` 对两段宽字符表达式做 RC4 解密。第一段恢复 `int(KEY[0:4])`，读取 Python 全局 `KEY` 的值 `6859`；随后由第 1、2、3 位更新 `g_State[3..6]`。正确状态为 `7a 6d cc 6f 79 64 7f cc`，于是后续解密表达式分别读取 password 第 9、10 位及 `KEY` 的切片。

将这些约束合并得到唯一的 12 字符输入：

```text
Hydr0ph11na3
```

通过四个检查后，Python 侧对 `FLAG` 做 Base64 解码，DLL 的 `Decrypt()` 使用上述 password 作为 RC4 key 原地异或，最后由 Python 拼成 `DUCTF{...}`。关键解密流程如下：

```python
flag = bytearray(base64.b64decode(FLAG))
buffer = (ctypes.c_byte * len(flag)).from_buffer(flag)
key = ctypes.create_string_buffer(password.encode("utf-8"))
get_helper().Decrypt(key, len(key) - 1, buffer, len(buffer))
```

### 验证

官方 `WRITEUP.md` 直接给出了同一 password；源码中的四个检查、RC4 解密入口及输出格式与之相互印证。本文据此静态整理，未执行 Python 手册、解压 DLL 或调用其中函数。

## 方法总结

- 核心技巧：把 Python `ctypes` 回调与 DLL 的原生状态机视为一条完整控制流，逐个消去字符约束。
- 识别信号：手册解压 DLL、DLL 又接受函数指针/回调时，检查可能借宿主语言的 globals 或 `eval`，不能只反编译 native 部分。
- 复用要点：先还原会影响后续 RC4 key 的状态字节，再解密动态生成的表达式；最后用源码中已有解密函数验证 recovered password。
