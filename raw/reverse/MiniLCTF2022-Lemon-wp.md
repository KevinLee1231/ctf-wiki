# MiniLCTF2022 Lemon Writeup

## 题目简述

附件 `task` 是 Lemon 语言的字节码。程序从固定种子生成 35 个伪随机数，再与 `res` 数组逐项异或并打印结果。题目的难点不在密码算法，而在识别 Lemon 字节码语义，以及避免用语义不等价的 Python 整数运算替代原运行时。

## 解题过程

根据随附语言文档还原关键指令：`const/store` 读写变量，`define` 定义函数，`class` 建类，`array/getitem` 操作数组，`jz` 与跳转构成循环，`bxor` 执行异或。整理后主逻辑为：

```javascript
var seed = 0xd33b470;

def next() {
    seed = (seed * 0xdeadbeef + 0xb14cb12d) % 0xffffffff;
    return seed;
}

var enc = [];
for (var i = 0; i < 35; i += 1) {
    enc.append(next());
}
for (var i = 0; i < 35; i += 1) {
    print(enc[i] ^ res[i]);
}
```

必须用出题时的 64 位 Linux Lemon 解释器运行原字节码或等价 Lemon 源码。若直接将同一表达式翻成 Python，Python 的任意精度整数会得到除首项外均非字节的结果；这不是正确 flag。原始仓库没有附解释器二进制，因此当前材料不能在本机独立重放运行时，但官方代码和多份赛后运行记录对输出一致：

```text
miniLctf{l3m0n_1s_s0_s0urrR77RrrR7}
```

这里保留 `miniLctf` 的原始大小写，不擅自规范化为 `miniLCTF`。

## 方法总结

自定义语言题首先要确认值类型、溢出、取模和位运算的真实语义。看似相同的源码在任意精度整数、定宽整数和解释器内部数值类型下可能产生完全不同的序列。可靠做法是先从字节码还原控制流，再在原运行时执行；若运行时缺失，应明确证据边界，并用多份相互独立的运行记录交叉验证结果，而不是伪造一次 Python“复现”。
