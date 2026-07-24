# YARV Abuse

## 题目简述

`YARV_Abuse` 以 Ruby Marshal 头 `04 08` 开始，反序列化后是 Ruby VM 指令序列的 `SimpleDataFormat` 数组。现代 Ruby 能执行 `Marshal.load`，但不再公开加载任意旧版 YARV 指令序列的接口，因此需要从数组中提取关键指令语义。

## 解题过程

指令先设置全局变量：

```text
$LOL = 1
```

随后定义方法 `ns(c)`。其效果可以简化为：

```ruby
def ns(c)
  $LOL.times { c += 1 }
  $LOL += 1
  c.chr
end
```

主程序依次以这些整数调用 `ns`：

```text
75, 77, 83, 65, 60, 78, 63, 65,
73, 73, 73, 71, 60, 57, 57, 68
```

由于 `$LOL` 从 1 开始逐次递增，等价于分别加上 1 到 16：

```python
values = [75, 77, 83, 65, 60, 78, 63, 65, 73, 73, 73, 71, 60, 57, 57, 68]
decoded = "".join(chr(value + index) for index, value in enumerate(values, 1))
print(decoded)
```

得到：

```text
LOVEATFIRSTSIGHT
```

后续 YARV 指令通过 `Array#send("*", "")` 连接字符，并把结果代入格式串 `UMDCTF-{%{&&&}}`，最终 flag 为：

```text
UMDCTF-{LOVEATFIRSTSIGHT}
```

其 SHA-256 与 README 中的 `f960207d97bbbe42ec14abc663d86b46837c53ba86e7a71e52a1c984953e7da7` 一致。

## 方法总结

字节码版本不兼容时，不必强行寻找完全相同的旧运行时。Marshal 数据仍保留了操作数、方法名、常量和调用顺序；只解释影响输出的少量指令，就能还原程序。这里的核心状态是递增的 `$LOL`，其余 VM 元数据都不是求解所必需。
