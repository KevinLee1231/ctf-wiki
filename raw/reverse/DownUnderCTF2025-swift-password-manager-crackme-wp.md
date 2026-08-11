# SwiftPasswordManager: CrackMe

## 题目简述

CrackMe 与 ClickMe 共用同一个 SwiftPasswordManager 手册，但目标改为在新建条目的 Password 文本框中输入正确字符串。`Binding.onChange` 每次密码字段更新都会调用 `checkCrackMePassword`；检查成功后程序直接弹窗 `Cracked! The flag is: <输入>`。

难点是 Swift 编译后 closure 的名字很长、反编译输出充满 `String.Index` 和引用计数调用。可用 `Binding.onChange` 的交叉引用定位到检查函数后，将其还原成字符位置、哈希和线性约束，不必逐行阅读庞大的反编译函数。

## 解题过程

### 还原输入边界和自定义哈希

检查函数首先要求：

```text
前缀：DUCTF{
后缀：.}
内部长度：29
```

对内部字符串 `password` 定义的 `h()` 从 `0x811c9dc5` 和 `0x01000193` 开始，逐字节执行 7 位循环左移异或与带索引的乘加，最终异或两个 64 位状态。函数同时比较 `password.h()` 的低 32 位和 `reversed(password).h()` 的高 32 位，因此候选必须同时满足正反两个方向的约束。

### 把 Swift 字符串操作化为普通约束

其余判断可以直接按源码重写为有限的字符约束：固定位置给出 `c`、`h`、`o`、`0`、`5`、`i`、`N`、`g`、`_` 等字符；对前 8 字符倒序隔位取样，得到 `g/i/0/h` 的关系；平方下标 $1,4,9,16,25$ 的单字符哈希固定；第 14、15、24 位满足三元一次方程组；两段四字符子串交错、倒序后应为 `hs_45p1_`；第 16--19 位交换相邻字符并加减 1 后应为 `q1^e`。

例如三元组使用：

$$
\begin{aligned}
x+2y+3z&=383\\
4x+5y+6z&=959\\
9x+8y+9z&=1641
\end{aligned}
$$

联立和位置约束即可唯一填满内部字符串。源码中的开发注释和题目配置给出的通过值为：

```text
DUCTF{cho05iNg_A_p4s5W0rd_15_h4Rd...}
```

把该字符串输入 Password 文本框会触发 on-change 检查并由应用自身显示成功提示。

### 验证

官方 WRITEUP 将入口定位为 `Binding.onChange(_:)`，题目共用手册中的 `CrackMe.swift` 则保留了完整的高层检查逻辑；题目 YAML 中的 flag 与该候选一致。本文没有运行应用或对 Swift 二进制执行动态调试。

## 方法总结

- 核心技巧：从 SwiftUI 的数据绑定回调而非冗长的 view-builder 符号进入，抽象出字符串约束求解器。
- 识别信号：Swift closure 反编译杂乱但持续出现 `hasPrefix`、`index(offsetBy:)`、`reversed`、`compactMap` 时，应转换为字符位置与序列变换，而不是追逐 retain/release。
- 复用要点：先固定长度/前后缀，再处理单字符和线性方程，最后用双向 hash 作整体校验；这样可减少枚举并避免 Swift `String` 索引噪声。
