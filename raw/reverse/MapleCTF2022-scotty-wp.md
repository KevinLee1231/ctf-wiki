# scotty

## 题目简述

题目给出一个 Haskell 可执行文件。源码使用高阶函数编码整数和列表：`I` 是带三个分支的 Scott 风格二进制自然数，`L` 是 Church/Scott 风格列表。程序把用户输入的每个字符编码后，与源码中一段极长的嵌套 lambda 项 `flag` 比较。

## 解题过程

先理解 `fromInt`：

```haskell
fromInt 0 = I $ \z s p1s -> z
fromInt n
  | even n    = I $ \z s p1s -> s    (unI (fromInt (n `div` 2)) z s p1s)
  | otherwise = I $ \z s p1s -> p1s  (unI (fromInt (n `div` 2)) z s p1s)
```

因此 `z` 表示 0，`s x` 表示 $2x$，`p1s x` 表示 $2x+1$。每个 `I $ \z s p1s -> ...` 从最内层 `z` 向外依次应用 `s/p1s`，就能恢复一个字符的 Unicode 码点。`L` 的 `n` 是空表，`c head tail` 是 cons；顺着外层连续的 `c` 即可逐字符遍历。

可以写一个简单解析器统计括号并把每段 `I` 交给递归求值，也可以给原 Haskell 文件增加：

```haskell
toInt = reI 0 (* 2) (\x -> 2 * x + 1)
toList = reL [] (:)
```

对 `flag` 执行 `map (chr . toInt) (toList flag)`，恢复：

```text
maple{7h4nk_y0u_4l0nz0_church.DwRVwXLKnlMQFnw5}
```

## 方法总结

高阶函数混淆通常仍有明确的构造器和消去器。找到 `fromInt/fromList` 后，最可靠的方法是写出它们的逆解释器，而不是手工化简上百层 lambda。这里 `s` 的名字容易让人误以为是普通后继函数，实际由 `n div 2` 可知它编码的是二进制低位 0；`p1s` 则编码低位 1。
