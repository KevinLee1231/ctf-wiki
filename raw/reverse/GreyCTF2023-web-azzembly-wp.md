# GreyCTF 2023 Web-Azzembly

## 题目简述

JavaScript 内嵌一段 WebAssembly 字节码，并按用户输入字符修改其中若干立即数。WASM 导出的 `check` 函数计算 82 条模 64 约束，返回不满足约束的数量；目标是让返回值为 0。恢复“输入字符到字节偏移”的映射后，可直接用 Z3 求整个线性同余系统。

## 解题过程

反汇编 WASM 后可以看到重复的指令片段：加载两个常量、相加、与 63、再与右端常量比较。每条约束等价于

$(x_i+x_j)\bmod64=c$，

少数约束只含一个变量。JavaScript 中的 `rep` 数组记录每个输入字符会写入哪些 WASM 字节；这些位置满足固定步长关系：

```python
equation_index = (byte_position - 37) // 11
```

把映射反转即可得到每条方程左侧涉及的字符。右端常量位于 WASM 数组的：

```python
rhs[i] = binary[46 + 11 * i]
```

为 82 个字符建立整数变量，将恢复出的单变量或双变量方程全部加入 Z3，并补上一条已知单变量锚点。求得模型后，对每个值取模 64，再按 JavaScript 使用的字符表还原字符：

```python
solver.add((x[i] + x[j]) % 64 == rhs)
solver.add(x[i] % 64 == rhs)  # 单变量约束
```

按下标排序模型结果，得到：

```text
grey{d1d_y0u_u53_4_c0nstr4int_s0lv3r_cuz_th1s_f1ag_1s_90nna_be_v3ry_lo0o0o00oo0ng}
```

将字符串交回 `chall.js` 后，WASM `check` 返回 0。

## 方法总结

题目的关键不是浏览器交互，而是识别高度重复的 WASM 指令模板和 JS 对二进制立即数的补丁规律。固定偏移与步长可把字节位置还原成方程编号；之后整个验证器就是稀疏线性同余约束，交给 SMT 求解比逐字符爆破可靠得多。
