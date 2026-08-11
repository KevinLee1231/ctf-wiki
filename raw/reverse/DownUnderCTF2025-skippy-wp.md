# skippy

## 题目简述

Windows 样本在解密 flag 前调用 `sandwich(key)` 与 `sandwich(iv)`，每次都把真正需要的 `decryptor()` 放在两个 `stone()` 调用之间。程序故意包含多个带醒目错误信息的崩溃点：`stone()` 尝试改写只读字符串常量，`decryptor()` 又解引用 `NULL`。若仅让程序“不崩溃”而跳过 `decryptor()`，AES key/IV 不会被还原，仍无法解密。

因此本题的决定性工作是根据崩溃字符串定位控制流、保留有用的右移变换、只绕过破坏性指令。

## 解题过程

### 区分故意崩溃与真正的解密步骤

源逻辑等价于：

```c
void sandwich(uint8_t *x) {
    stone(x);       // 改写 const 字符串，故意崩溃
    decryptor(x);   // 先 NULL 解引用，后逐字节右移，右移是必需的
    stone(x);       // 再次故意崩溃
}
```

发布二进制中可以从 `"Oh no! Skippy is about to trip!"` 和 `"Uh oh... Skippy sees a null zone"` 两个字符串定位这些函数。对二进制做最小 patch：跳过 `stone()` 的调用/写常量指令，并 NOP `decryptor()` 中 `*NULL` 的读取；不要跳过其后循环。

### 恢复 AES key/IV 并解密

`decryptor()` 的保留循环计算 `x[i] = x[i] >> 1`。题目生成脚本表明，内嵌 key 与 IV 是对原文逐字节左移一位得到的，所以右移后还原为：

```text
key = "skippy_the_bush_"
iv  = "kangaroooooooooo "
```

注意 IV 末尾包含空格；AES key/IV 都取前 16 字节。随后 `decrypt_bytestring()` 用 AES-128-CBC 对 `precomputed` 解密并打印结果。第三个 `stone(decrypted)` 也必须绕过，否则解密完成前仍会因改写字符串常量而退出。

### 验证

官方题解的验证标准是“正确 patch 后程序自行解密并输出 flag”。`bitshift.py` 给出原始 key/IV 与左移生成逻辑，`main.c` 给出与之对应的右移及 AES-CBC 调用。本文没有改写或运行发布样本。

## 方法总结

- 核心技巧：按崩溃字符串恢复调用图，最小化 patch 以保留夹在反分析崩溃点中的真实变换。
- 识别信号：同一函数前后反复出现故意 fault，而中间有针对 key/IV 的字节运算时，不要把整个函数都跳过。
- 复用要点：patch 后的关键验证是解密路径仍执行并给出可读输出；还原对称加密参数时须保留空格等不可见的有效字节。
