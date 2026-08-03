# Without a Trace

## 题目简述

服务把 25 字节 Flag 切成五个 5 字节大端整数 $f_0,\ldots,f_4$。用户只能提交对角矩阵 $M=\operatorname{diag}(u_0,\ldots,u_4)$；在要求 $\det(M)\ne0$ 后，服务返回 $\operatorname{tr}(FM)=\sum f_i u_i$。因此不能把其余系数设为零逐项读取，但可以在一次线性组合中无进位地打包全部秘密。

## 解题过程

源码的 `check` 遍历所有排列并按逆序数决定符号，实际计算的就是行列式。对角矩阵的行列式为 $\prod u_i$，所以只要五个输入都非零就能通过。

每个 $f_i$ 来自 5 字节，满足 $0\le f_i<256^5<10^{13}$。选择更大的进制 $B=10^{16}$，提交

$$
(u_0,u_1,u_2,u_3,u_4)=(B^0,B^1,B^2,B^3,B^4).
$$

返回值就成为无进位的混合进制打包：

$$
R=f_0+f_1B+f_2B^2+f_3B^3+f_4B^4.
$$

无需依赖十进制字符串切片，直接用整除和取模逐块提取更稳妥：

```python
from Crypto.Util.number import long_to_bytes

B = 10**16
inputs = [B**index for index in range(5)]
# 新建连接，依次提交 inputs，解析服务最后返回的整数为 response。

chunks = [(response // (B**index)) % B for index in range(5)]
flag = b"".join(long_to_bytes(chunk, 5) for chunk in chunks)
print(flag.decode())
```

指定长度 5 的转换能够保留块开头可能出现的零字节。官方实例返回值拆出的五块依次为 `uiuct`、`f{tr4`、`c1ng_`、`&&_mu`、`lt5!}`，组合得到：

```text
uiuctf{tr4c1ng_&&_mult5!}
```

## 方法总结

- 非零行列式约束只禁止直接选择坐标轴，并未阻止通过位置制编码把多个秘密系数装入一次响应。
- 基数必须严格大于每个秘密块的上界，否则相邻位会发生进位，无法独立恢复。
- 使用整数运算 `(R // B**i) % B` 比固定宽度字符串切片更通用；还原固定字节块时也应保留前导零。
