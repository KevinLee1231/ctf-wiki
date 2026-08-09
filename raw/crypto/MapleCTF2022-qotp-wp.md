# qotp

## 题目简述

服务把 flag 的每一位编码为二维实数“量子态”，并允许用户为每次查询提供一组测量基。服务器每次都会随机旋转编码基，因此单次测量不能确定明文；但明文在多次查询间固定，而且所用模型只有实平面上的二维旋转，并没有真正隐藏掉统计偏差。

## 解题过程

若直接沿题目随机基测量，结果接近随机。选择角度 $-\pi/4$ 的固定正交基：

$$
\begin{pmatrix}
\cos\theta & \sin\theta\\
-\sin\theta & \cos\theta
\end{pmatrix},\qquad \theta=-\frac{\pi}{4}.
$$

对编码 0 和编码 1 而言，随机旋转虽然会扰动单次结果，但在这个方向上的投影概率具有相反的稳定偏差。于是重复请求同一密文，对每个比特位置独立计数，按多数票恢复：

```python
samples = 333
count = [0] * 352

for _ in range(samples):
    result = query_measurement(-pi / 4)
    for i, bit in enumerate(result):
        count[i] += bit == ord("1")

bits = "".join("1" if n > samples // 2 else "0" for n in count)
flag = long_to_bytes(int(bits, 2))
```

333 次不是严格下限，而是在请求量和多数票置信度之间的实用选择。恢复得到：

```text
maple{ru1n3d_BY_th0SE_d412n_r0t4ti0Ns_4ga1N}
```

## 方法总结

随机化并不自动等于信息论安全。只要同一秘密可被重复采样，而且不同明文在攻击者选择的观测基上保留非零统计距离，就可以通过多数票放大微小偏差。实际脚本应同时输出每一位票数到半数的最小距离，用它判断是否需要增加采样，而不是只相信一次解码结果。
