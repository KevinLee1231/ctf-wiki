# sillygoose

## 题目简述

服务在 $[0,10^{100}]$ 中随机选择整数，并只回答猜测过大、过小或正确。500 次限制足以二分查找，但 10 秒时间限制要求自动交互。

## 解题过程

维护闭区间并根据响应缩小一半：

```python
low, high = 0, 10 ** 100
while low <= high:
    mid = (low + high) // 2
    send(str(mid))
    response = recvline()
    if "too large" in response:
        high = mid - 1
    elif "too small" in response:
        low = mid + 1
    else:
        break
```

最坏查询次数约为：

$$\lceil\log_2(10^{100}+1)\rceil=333,$$

低于 500 次。猜中后得到：

```text
n00bz{y0u_4r3_4_sm4rt_51l1y_g0053}
```

## 方法总结

单调的大小反馈就是比较 oracle，二分查找是最优策略之一。大整数应直接用语言的任意精度整数处理，避免浮点中点造成精度丢失。
