# roulette

## 题目简述

程序表面上要求输入轮盘数字，实际把整行输入按 4 字节小端分块，并用中国剩余定理、一个小型 VM 和滚动状态逐块校验。大量 AI 拒绝文本、Base64 字符串和假轮盘输出只是诱饵，不参与最终 `verify_flag`。

## 解题过程

对第 $i$ 个块，先还原两个余数：

$$
\begin{aligned}
r_1&=\text{R1\_OBF}[i]\oplus(\operatorname{fold1}(\text{SEED},i)\bmod m_1),\\
r_2&=\text{R2\_OBF}[i]\oplus(\operatorname{fold2}(\text{SEED},i)\bmod m_2).
\end{aligned}
$$

$m_1,m_2$ 互素且乘积大于 $2^{32}$，用 CRT 唯一恢复 32 位 $k_i$。以 $k_i$、当前滚动状态 $s$ 和块编号 $i$ 初始化 8 个 VM 寄存器，解释 `VM_PROG` 中的 `MOV`、`XOR`、`ADD`、`MUL`、`ROTL`、`SHR`，最终寄存器 6 是掩码。

校验式为：

$$
x_i\oplus\operatorname{vm\_mask}(k_i,s,i)=\text{ENC}[i],
$$

因此可以直接求出：

$$
x_i=\text{ENC}[i]\oplus\operatorname{vm\_mask}(k_i,s,i).
$$

恢复每个 $x_i$ 后，必须按程序原式更新滚动状态：

```python
s = rotl32(s ^ x ^ ((k * 0x045D9F3B) & 0xffffffff), (i % 5) + 7)
s = (s + 0x9E3779B9 + i * 0x7F4A7C15) & 0xffffffff
```

把所有 $x_i$ 转成 4 字节小端并拼接，按 `FLAG_LEN` 截断，即得：

```text
UMDCTF{I_R3ALLY-want-to-pl4y-the-p0werball,+but-my-d4d-said-no-so-im-b3tting-ill-win-on-POLYMARKETinstead}
```

## 方法总结

验证函数逐块给出了可逆关系：余数去混淆后用 CRT 恢复 key，VM 只生成异或掩码，密文数组因此直接泄露明文块。唯一需要顺序执行的是滚动状态；若字节序或某轮状态更新错误，后续所有块都会失真。
