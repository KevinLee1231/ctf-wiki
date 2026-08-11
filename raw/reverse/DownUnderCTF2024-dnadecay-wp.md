# DNAdecay

## 题目简述

`dna.rb` 原本是一个 Ruby 程序，使用 `doublehelix` 将 `puts"..."` 形式的源码转换为由 `A`、`T`、`G`、`C` 和 `-` 组成的双螺旋排版。出题脚本随后以概率把这些字符替换为空格。难点不是运行 Ruby，而是利用该编码的行结构恢复被破坏的程序语义，故归入 Reverse。

## 解题过程

先检查生成器。原始逻辑为：

```ruby
dna = doublehelix('puts"DUCTF{...}"')
decayed_dna = randomly_replace(dna, "A", " ")
decayed_dna = randomly_replace(decayed_dna, "T", " ")
decayed_dna = randomly_replace(decayed_dna, "G", " ")
decayed_dna = randomly_replace(decayed_dna, "C", " ")
decayed_dna = randomly_replace(decayed_dna, "-", " ")
```

因此每一行原本只能属于 `AT`、`TA`、`GC`、`CG` 四种配对之一，连字符仅决定两端的水平位置。官方求解器逐行去掉空格，按两端已保留的碱基和连字符方向恢复唯一配对：例如左端只剩 `A` 时补为 `AT`，右端只剩 `C` 时补为 `GC`。完全为空的行记录为 `missing`。

将确定行复原后，候选源码已能解出大部分字符串。对剩余 8 个 `missing` 位置枚举四种配对，筛掉使 `doublehelix` 还原结果不可打印的候选；再以题目文本的固定短语 “The Mitochondria is the power house of the cell” 约束歧义。得到完整输出：

```
DUCTF{7H3_Mit0cHOndRi4_15_7he_P0wEr_HoUsE_of_DA_C3LL}
```

## 方法总结

随机擦除的程序性编码不应直接把空格当作无意义填充。先从生成器确定每个符号单元的合法集合，再以几何布局恢复大多数确定位置；仅对信息论上确实缺失的位置做很小的枚举。最后使用语法、可打印输出和上下文短语多重校验，比盲目恢复整段源码可靠得多。
