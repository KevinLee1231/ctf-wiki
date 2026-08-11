# DownUnderCTF 2023 My First C Program Writeup

## 题目简述

附件虽然扩展名为 `.c`，实际使用的是恶搞语言 DreamBerd，而不是 C。程序无需编译；只要按照该语言的非常规优先级、数组下标和 `previous` 语义，追踪传给 `print_flag` 的五个参数即可。

## 解题过程

最终调用为：

```text
print_flag(thank, vars[-1], end, heck_eight, ntino)
```

而输出模板是：

```text
DUCTF{${start}_${realstart}_${end}_${secondmiddle}_1s_${middle}_C}
```

逐项还原变量：

- `thonk(1, 'n')` 中，感叹号数量影响优先级，实际返回 `ret`，即 `Th1nk`，所以 `thank = "Th1nk"`。
- DreamBerd 数组从 `-1` 开始索引，因此 `vars[-1]` 取得首项 `R34L`。
- `looper()` 返回 15，故外层插值得到 `end = "th15"`。
- `get_a_char()` 先把字符改为 `I`，后面又改为 `A`，但 `return previous dank_char` 返回前一个值，所以 `heck_eight = "I"`。
- `math()` 返回 $10\bmod5=0$。
- `guesstimeate()` 利用 `previous` 返回先前的 `guess`，结果为 `nT`，因此 `ntino = "D0nT"`。

参数到形参的映射为：

```text
end          <- thank       = Th1nk
middle       <- vars[-1]    = R34L
secondmiddle <- end         = th15
start        <- heck_eight  = I
realstart    <- ntino       = D0nT
```

代入模板得到：

```text
DUCTF{I_D0nT_Th1nk_th15_1s_R34L_C}
```

## 方法总结

文件扩展名和题面中的“C”都是误导。识别 DreamBerd 后，不必实现完整解释器，只需围绕最终输出做程序切片，掌握本题实际用到的几条语言规则。对 esolang 逆向而言，追踪决定输出的最小语义集合通常比尝试运行全部程序更高效。
