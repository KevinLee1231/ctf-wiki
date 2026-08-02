# wt-two

## 题目简述

程序让 `main` 同时承担输入验证和递归 Fibonacci 计算。当 `argc==1` 时读取至少 30 字节输入；验证第 $i$ 个字符时，它把整数 $i$ 放进伪造的 `argv[0]` 缓冲区，再调用 `main(3, strs)`。`argc==3` 分支从 `argv[0]` 读回整数并递归计算，基例 $F(0)=F(1)=1$。

验证关系为：

$$
input_i=message_i\oplus F(i).
$$

## 解题过程

虽然源码用递归自调用制造混淆，实际密钥序列就是偏移一位定义的 Fibonacci 数列。把二进制中 30 个 `message` 整数抄出，迭代生成同样的 $F(i)$ 后逐项异或即可：

```python
message = [
    117, 107, 97, 119, 99, 115, 122, 97, 15, 67,
    49, 245, 196, 269, 533, 948, 1618, 2679, 4154, 6658,
    10915, 17756, 28613, 46360, 75060, 121457, 196390,
    317717, 514246, 832085,
]

fib = [1, 1]
while len(fib) < len(message):
    fib.append(fib[-1] + fib[-2])

print("".join(chr(value ^ key) for value, key in zip(message, fib)))
```

输出恰好是 30 字节 flag：

```text
tjctf{wt-the-twoooooas48%@dfs}
```

也可以直接把该字符串输入程序，所有 30 次比较均通过并打印 `Nice!!!`。

## 方法总结

- 递归调用 `main` 和伪造 `argv` 只是控制流混淆；先按 `argc` 拆分两种语义，就能看出独立的 Fibonacci 函数。
- 注意本题基例是 $F(0)=F(1)=1$，不是常见的 $F(0)=0,F(1)=1$。
- `message` 使用 `int` 保存是为了容纳增长的 Fibonacci 值；应先按整数异或，再转字符，不能预先截断每个常量。
