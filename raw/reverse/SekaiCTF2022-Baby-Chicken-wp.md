# Baby Chicken

## 题目简述

附件是由 CHICKEN Scheme 编译的程序，同时保留了 Scheme 源码和一个 C 辅助函数。程序要求输入恰好 26 个字符，并将每个字符与位置相关的欧拉函数值异或，再和固定整数数组比较。

校验过程没有不可逆步骤。只要从 Scheme 层找到常量数组，再从 C 层还原 `chick(i)`，逐字节异或即可恢复 flag。

## 解题过程

`chicken.scm` 通过 FFI 导入 C 函数：

```scheme
(define babychicken (foreign-lambda int "chick" int))
(define arr
  '(59 113 71 25 9 123 101 81 80 99 45 95
    13 59 24 9 21 91 83 11 92 3 28 113 81 45))
```

输入长度必须为 26。第 $i$ 个位置的检查是：

```scheme
(= (nth i arr)
   (bitwise-xor
       (babychicken i)
       (char->integer (nth i input))))
```

也就是：

$$
\text{arr}[i]=\operatorname{ord}(\text{input}[i])\oplus\operatorname{chick}(i)
$$

异或是自身的逆操作，所以原字符为：

$$
\operatorname{ord}(\text{input}[i])=\text{arr}[i]\oplus\operatorname{chick}(i)
$$

`util.c` 给出了 `chick()` 的完整定义：

```c
int f(int n) {
    return n * n + 1337 * n + 31337;
}

int chick(int n) {
    return phi(f(n + 42)) % 128;
}
```

其中 `phi()` 计算欧拉函数 $\varphi(n)$。按原式复现即可：

```python
arr = [
    59, 113, 71, 25, 9, 123, 101, 81, 80, 99, 45, 95, 13,
    59, 24, 9, 21, 91, 83, 11, 92, 3, 28, 113, 81, 45,
]

def phi(n):
    result = n
    factor = 2
    while factor * factor <= n:
        if n % factor == 0:
            while n % factor == 0:
                n //= factor
            result -= result // factor
        factor += 1
    if n > 1:
        result -= result // n
    return result

def chick(index):
    value = index + 42
    return phi(value * value + 1337 * value + 31337) % 128

flag = "".join(
    chr(target ^ chick(index))
    for index, target in enumerate(arr)
)
print(flag)
```

输出为：

```text
SEKAI{5ch3m3_Ch1cK_K4w411}
```

## 方法总结

CHICKEN Scheme 编译产物看起来不如普通 ELF 直观，但题目保留的 `.scm` 与 C 文件已经把跨语言边界完整暴露出来。先从 Scheme 中确认输入长度、常量数组和异或关系，再跟进 FFI 所调用的 `chick()`，比直接逆向编译后的运行时框架高效得多。

欧拉函数本身不需要暴力枚举互素数；分解质因数后使用 $\varphi(n)=n\prod_{p\mid n}(1-1/p)$ 即可快速计算。最后逐字节正向代回常量数组，可以确认恢复结果没有索引偏移。
