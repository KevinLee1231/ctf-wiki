# UMDCTF 2018 - Speed Demon

## 题目简述

附件是 Ruby 程序。外层脚本没有直接保存普通 Ruby 源码，而是把一份 YARV 指令序列以 `RubyVM::InstructionSequence` 的序列化形式嵌入字符串，再通过 Fiddle 调用内部加载函数执行。

## 解题过程

`speed_demon.rb` 中的大字符串 `LFDY` 以：

```text
YARVInstructionSequence/SimpleDataFormat
```

开头，说明它是 Ruby VM 字节码。外层的 `V.e` 通过动态拼接函数名，最终调用 `rb_iseq_load`，再执行反序列化出的指令序列。

仓库保留了对应的开发文件 `dev/inner.rb` 和 `dev/inner.rb.vm`。字节码中的常量赋值依次为：

```text
U M D C T F - { N E S T E D B Y T E C O D E }
```

其中变量之间还有引用，例如 `q6 = q3`、`q10 = q5`，按赋值关系还原后连接即可得到：

```text
UMDCTF-{NESTEDBYTECODE}
```

其 SHA-256 为：

```text
1b19fdd4377b6edecaebbaee5a6325c3328ded68833e7989b80a9bb07b6f22e5
```

与仓库 `README.md` 一致。

## 方法总结

“加速 Ruby”是对预编译 VM 字节码的提示。遇到动态拼接、Fiddle 和 `rb_iseq_load` 时，不必先完全去混淆所有包装代码；先识别序列化格式，再反汇编或读取同仓库的原始指令序列，能够直接恢复常量和控制流。
