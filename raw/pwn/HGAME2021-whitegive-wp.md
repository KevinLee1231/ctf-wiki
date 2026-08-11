# whitegive

## 题目简述

程序读取一个 `unsigned long long`，却用它与字符串字面量 `"paSsw0rd"` 直接执行 `==` 比较。C 中的字符串字面量在表达式里会退化为指针，因此这里比较的不是字符串内容，而是输入整数与该字面量地址是否相等；比较成立后程序执行 `system("/bin/sh")`。

## 解题过程

核心源码等价于：

```c
unsigned long long num;

scanf("%ld", &num);
if (num == "paSsw0rd") {
    system("/bin/sh");
}
```

在 IDA 中查看比较位置，可以看到编译器通过 `lea rax, aPassw0rd` 取得字面量地址。数据段中 `aPassw0rd` 位于 `0x402012`，转换为程序要求的十进制为 `4202514`。

连接服务后输入：

```text
4202514
```

比较成立后获得 shell，再读取当前目录中的 `flag` 文件：

```text
hgame{W3lCOme_t0_Hg4m3_2222Z222zO2l}
```

官方 PDF 只写到 getshell，这一服务器输出由 [MiaoTony 的同期复盘](https://miaotony.xyz/2021/02/07/CTF_2021HgameWeek1/) 补齐；漏洞成因和输入值已经在正文中完整保留。

## 方法总结

本题利用的是“指针比较被误当作字符串比较”。看到字符串与整数直接使用 `==` 时，应检查反编译结果和实际类型；在 C 中比较字符串内容应使用 `strcmp`。如果二进制未开启 PIE，字符串地址固定，可以直接把该地址作为数值输入；开启 PIE 后则还需要先获得程序基址。
