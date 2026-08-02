# TSGCTF2024 H*: A Refinement Type System for Haskell

## 题目简述

服务接收一种玩具语言 H*。同一 AST 会被翻译为两份程序：

1. 生成 F*，用 refinement type 检查安全性。
2. 若 F* 验证通过，再生成 Haskell、用 GHC 编译并执行。

F* 侧预置：

```fstar
val flag: x:int{false} -> ML unit
let flag _ = print_string "flag{hello}\n"
```

也就是说，`flag` 要求一个满足 `false` 的整数，正常程序不可能构造。Haskell 侧却只生成普通函数：

```haskell
flag :: Integer -> IO ()
flag x = putStrLn "<real flag>"
```

目标是让 F* 接受程序，同时让生成的 Haskell 真正调用 `flag`。

## 解题过程

### 1. 构造“若返回则证明 false”的发散函数

H* 支持 `Dv` effect，表示函数允许不终止。定义：

```text
let rec loop:: x::Integer{1>0} -> Dv(x::Integer{0>1}) =
  \x -> loop x
```

输入 refinement `1>0` 恒真，返回 refinement `0>1` 恒假。这个函数永远递归，不会实际产生违反类型的返回值；F* 的 call-by-value 语义允许把发散计算声明为这样的空返回类型。

### 2. 利用 F* 的 call-by-value 推导

提交完整程序：

```text
let rec loop:: x::Integer{1>0} -> Dv(x::Integer{0>1}) = \x -> loop x in
  let n = loop 1 in
  flag 1
__EOF__
```

在 F* 的 call-by-value 模型中，执行 `let n = loop 1 in ...` 前必须先求值 `loop 1`。若控制流真的到达后续表达式，就意味着得到了一个类型为 `Integer{0>1}` 的值 `n`；该上下文包含矛盾，因此可由 false 推出任意命题，包括把字面量 1 当成 `x:int{false}` 传给 `flag`。验证器据此接受程序。

实际 F* 程序不会越过绑定，因为 `loop 1` 无限递归，所以从其执行语义看 `flag` 确实不可达。

### 3. Haskell 的惰性求值跳过发散绑定

Haskell 生成器会丢弃 refinement 和 effect，只保留：

```haskell
loop = \x -> loop x

main =
  let n = loop 1
  in flag 1
```

Haskell 使用 call-by-need。局部绑定 `n` 从未被后续表达式引用，所以 `loop 1` 的 thunk 根本不会求值；程序直接执行 `flag 1`。服务输出：

```text
TSGCTF{prov1ng_50undness_of_syst3ms_1s_b0ring}
```

## 方法总结

本题利用的是验证语言与执行语言的求值策略不一致。F* 的证明建立在 call-by-value 上：发散绑定之后的代码不可达，空 refinement 可安全存在；翻译到惰性 Haskell 后，未使用的发散绑定被跳过，原本“不可达”的调用变为可达。只验证一种语言再执行另一种语言时，编译器必须证明翻译保持语义，尤其要统一求值顺序、effect、严格性和异常行为；简单删除 refinement 注解并不能保持 F* 的安全结论。
