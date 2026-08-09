# Numbers 2

## 题目简述

服务在限时内随机提出 100 道数论计算题，类型只有最大质因数、最大公约数和最小公倍数。主障碍是解析并快速自动应答。

## 解题过程

根据提示文本分流：

```python
import math

def largest_prime_factor(n):
    factor = 2
    largest = 1
    while factor * factor <= n:
        while n % factor == 0:
            largest = factor
            n //= factor
        factor += 1
    return max(largest, n)

# greatest common divisor -> math.gcd(x, y)
# least common multiple   -> math.lcm(x, y)
```

逐轮读取问题、提取整数并立即发送对应结果。答对 100 轮后得到：

服务端把质因数题的随机范围写成从 0 开始；若抽到 `0` 或 `1`，其 `sympy.primefactors(x)[-1]` 会在校验阶段自身报错，这一实例无法正常完成，应重新连接，而不是伪造一个“最大质因数”。

```text
n00bz{numb3r5_4r3_fun_7f3d4a}
```

## 方法总结

交互计算题应先按稳定关键词路由，再调用确定性算法。不要依赖官方脚本中的固定字符串切片位置；用正则提取数字更能适应提示长度变化。
