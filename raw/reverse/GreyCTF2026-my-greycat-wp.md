# my-greycat

## 题目简述

题目给出一个约 51 MB 的 64 位 ELF `main`。它把内嵌的两个等长 `uint32_t` 数组 `cs`、`ds` 逐项处理，输出约 6.4 MB 的 MOV 文件。源码说明每个输出字节由初值 1 经 `ds[i]` 次循环相乘得到：

$$
b_i=c_i^{d_i}\bmod257.
$$

生成器实际选择的 $d_i$ 与 $256$ 互素，令 $e_i=d_i^{-1}\pmod{256}$，再将原视频字节 $p_i$ 加密为 $c_i=p_i^{e_i}\bmod257$。所以二进制慢循环的结果正是原始 $p_i$。解题要点是从 ELF 的符号表和数据段定位数组、将逐次乘法改写为快速模幂；模运算本身非常直接，决定性主障碍是二进制数据布局还原，归为 reverse。

## 解题过程

### 定位数组而不执行慢循环

发布二进制没有剥离符号，`cs`、`ds` 和 `n` 仍在 ELF symbol table。要注意 symbol value 是虚拟地址，不能直接当作文件偏移：应先解析 ELF64 section header，再找满足 `section.addr <= value < section.addr + section.size` 的 section，并换算：

$$
\text{fileOffset}=\text{section.offset}+(\text{symbolVA}-\text{section.addr}).
$$

读取 `n` 后，将 `cs` 与 `ds` 各按连续的 4 字节 little-endian `uint32_t` 解包。官方 `fast_decrypt.py` 完整实现了 ELF64 symbol-table 解析、VA 到文件偏移转换与范围检查，因此不会依赖某个固定编译地址或手写偏移。

### 快速还原 MOV

原程序的内层循环相当于 `pow(c, d, 257)`，但直接执行每个大指数的循环会极慢。Python 的三参数 `pow` 使用快速模幂，可把每项替换为：

```python
for i in range(n):
    c = struct.unpack_from("<I", blob, cs_off + 4 * i)[0]
    d = struct.unpack_from("<I", blob, ds_off + 4 * i)[0]
    output[i] = pow(c, d, 257) & 0xff
```

运行题目提供的优化 solver：

```bash
python fast_decrypt.py ../dist/main -o data.MOV
```

它可按 chunk 并行处理数组，但单线程结果与多线程结果应相同；这里的 `& 0xff` 是因为模数为 257，而原视频 byte 落在 0 至 255。

### 验证

输出应是可播放的 MOV，而非仅凭可读字符串判断。仓库中的 `solve/data.MOV` 与生成端保留的 `source/greyctf-trailer-modified.MOV` 的 SHA-256 都是：

```text
19967f37e79c34a05a625525e062dd579fcd3f2dffcc872bdb3cb9ca6692e7c5
```

视频中给出的 flag 为：

```text
grey{d1d_y0u_s33_mY_pr3sid3nT?}
```

## 方法总结

- 核心技巧：遇到“重复乘法取模”的二进制循环时，先将其化为模幂，再以快速幂替代逐次迭代。
- 识别信号：ELF 体积异常大、源码只有数组驱动的单字节输出循环，且符号未剥离时，优先恢复数据段中的表而不是运行原程序等待结果。
- 复用要点：解析 ELF 时要做 VA 到文件偏移换算并核验数组范围；二进制载体解密后，使用格式可播放性或 hash 与已知生成物交叉验证。
