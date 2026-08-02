# TSGCTF2020 Self Host WP

## 题目简述

附件定义了高级语言 X、栈式汇编 Y、X 解释器、Y 模拟器，以及一个用 X 编写且能把 X 编译为 Y 的自举编译器 `compiler.x`。服务要求提交一个 Y 编译器，并依次检查：

1. 它能正确编译随机 X 测试；
2. 它编译官方 `compiler.x` 后得到的第二代编译器仍正确；
3. 第二代编译器再次编译 `compiler.x` 时输出完全不变。

最后服务把完整 flag 嵌入一段只输出 `flag[0]` 的 X 程序，再交给通过检查的编译器。提交诚实编译器只能看到首字节；目标是构造能自我传播的恶意编译器，即 Ken Thompson 的 trusting-trust compiler backdoor。

## 解题过程

先用官方自举编译器生成一个正常 Y 编译器：

```bash
python3 interpreter.py compiler.x < compiler.x > compiler.y
```

它足以通过功能测试并得到首字节，但服务随后编译的是官方源码，普通修改会在下一代消失。恶意编译器 $B$ 需要满足三条规则：

- 输入以 token `flag` 开头时，识别服务构造的 flag 程序，直接生成输出完整字符串的 Y 汇编；
- 输入是官方编译器 $A$ 时，输出 $B$ 自身的汇编，使后门跨代传播且满足 `code2 == code3`；
- 其他输入完全按照 $A$ 编译，继续通过随机测试。

仓库的 `compiler_exploit.x` 在原编译器 `main` 的 tokenization 后加入两个分支。flag 分支从 token `s[8]` 取得字符串常量并生成：

```text
makelist #0 <完整flag字节列表>
mov sp 1
write
hlt
```

官方 `compiler.x` 的首个函数名是 `error`，因此第二个分支用 `s[0] == "error"` 识别“正在编译编译器自身”，并输出恶意编译器的完整 Y 汇编。

自我输出不能直接在源码中无限嵌套自身，需采用 quine 的“头部 + 头部数据 + 中间逻辑 + 尾部数据 + 尾部”结构。修改版源码先用两个哨兵列表占位：

```text
head = [123,456,789]
tail = [314,159,265]
```

先让官方编译器把 `compiler_exploit.x` 编译成临时汇编 `tmp.s`。`generator.py` 在两个哨兵处切分汇编，把真实 head、tail 的每个字符转成整数列表，再替换哨兵，得到最终 `exploit.s`。运行时 self-host 分支执行：

```text
head + list2str(head) + 中间固定汇编 + list2str(tail) + tail
```

因此输出恰好等于自身，第二、三代字节完全一致。服务最后构造：

```c
flag;
main(){ flag = "<完整flag>"; write([flag[0]]); }
```

恶意编译器命中 flag 分支，忽略原来的只输出首字节逻辑，生成输出整个字符串的汇编，得到：

```text
TSGCTF{You_now_understand_how_Ken_Tompson_Hack_works}
```

题目官方 flag 中把 Thompson 拼成了 `Tompson`，复现时应保留仓库中的实际字符串。

## 方法总结

本题要求理解自定义语言、VM、编译器代际关系与 quine，因此归入 Reverse。核心不是篡改一次编译结果，而是让恶意行为成为固定点：编译普通程序时保持兼容，遇到目标程序时植入后门，遇到编译器源码时再生自身。验证供应链时，只比较编译器能否通过功能测试或能否自举并不足够；trusting-trust 攻击可以同时满足这些性质，必须引入独立工具链、多样化双重编译或可验证构建来打破单一编译器的信任闭环。
