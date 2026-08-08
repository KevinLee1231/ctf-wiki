# 0.s1gn1n

## 题目简述

题目给出一个 32 位原生程序。程序先用一组恒真分支和一个不可达字节干扰反编译，再依次对输入执行树的中序遍历、标准 Base64 编码和相邻字节异或，最后将结果与内置数组结合并求和校验。

决定性障碍是恢复校验链中各层变换的先后顺序，并把树遍历造成的位置置换逆回来，因此归类为 Reverse。

## 解题过程

### 清除恒真分支

从输入提示字符串定位到校验函数附近，可以看到如下指令：

```text
.text:00401633  jz   short loc_401638
.text:00401635  jnz  short loc_401638
.text:00401637  db   0C7h
.text:00401638  loc_401638:
```

`jz` 与紧随其后的 `jnz` 无论标志位为何都会跳到同一地址，所以中间的 `0xC7` 不会执行，只是用于破坏反汇编边界。将其识别为花指令后，校验函数的主干就很清楚了。

### 还原校验链

核心逻辑可整理为：

```c
memset(traversed, 0, 64);
root = build_tree(input);
inorder(root, traversed, &length);
encoded = base64_encode(traversed, strlen(traversed), &encoded_length);

for (i = encoded_length - 1; i != 0; --i) {
    encoded[i] ^= encoded[i - 1];
    encoded[i] ^= answer[i];
}

result = -28;
for (j = 0; j < encoded_length; ++j)
    result += (signed char)encoded[j] - 1;
return result;
```

循环按从后向前的顺序修改数据，因此每个位置使用的仍是原始 Base64 前一字节。正确输入应满足

$$
answer_i = b_i \oplus b_{i-1}, \qquad 1 \le i < n,
$$

其中 $b$ 是 Base64 字节串。第 0 字节没有进入异或循环，仍为 `X`，即十进制 88。本题编码长度为 60，所以最终求和为

$$
-28 + 88 - 60 = 0,
$$

恰好通过校验。这也解释了看似特殊的初值 `-28`。

反解时从前向后做累计异或：

$$
b_0=answer_0,\qquad b_i=answer_i\oplus b_{i-1}.
$$

所得 Base64 串为：

```text
X1JLRjFfbmlkZ197MG5GaV9pQGVycnRMfTNzM21ucmlDZ2VubkV2X1RJRXM=
```

标准 Base64 解码得到的是中序遍历后的字符序列，而不是原 flag：

```text
_RKF1_nidg_{0nFi_i@errtL}3s3mnriCgennEv_TIEs
```

程序构树与中序遍历对应的两个唯一字符序列分别是：

```text
原位置序列：0123456789abcdefghijklmnopqrstuvwxyzABCDEFGH
遍历位置序列：vfw7xgy3zhA8BiC1DjE9FkG4Hlam0nbo5pcq2rds6teu
```

对原位置序列中的每个字符，在遍历位置序列中找到相同字符所在的下标，再从解码结果取该下标的字节，即可逆转位置置换。

### 完整验证脚本

```python
import base64

answer = [
    0x58, 0x69, 0x7B, 0x06, 0x1E, 0x38, 0x2C, 0x20,
    0x04, 0x0F, 0x01, 0x07, 0x31, 0x6B, 0x08, 0x0E,
    0x7A, 0x0A, 0x72, 0x72, 0x26, 0x37, 0x6F, 0x49,
    0x21, 0x16, 0x11, 0x2F, 0x1A, 0x0D, 0x3C, 0x1F,
    0x2B, 0x32, 0x1A, 0x34, 0x37, 0x7F, 0x03, 0x44,
    0x16, 0x0E, 0x01, 0x28, 0x1E, 0x68, 0x64, 0x23,
    0x17, 0x09, 0x3D, 0x64, 0x6A, 0x69, 0x63, 0x18,
    0x18, 0x0A, 0x15, 0x70,
]

for i in range(1, len(answer)):
    answer[i] ^= answer[i - 1]

traversed = base64.b64decode(bytes(answer))
original_order = b"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGH"
inorder_order = b"vfw7xgy3zhA8BiC1DjE9FkG4Hlam0nbo5pcq2rds6teu"

flag = bytes(traversed[inorder_order.index(ch)] for ch in original_order)
print(flag.decode())
```

运行结果为：

```text
miniLCTF{esrevER_gnir33nignE_Is_K1nd_0F_@rt}
```

## 方法总结

这道题的关键不是单独识别 Base64，而是确认整条数据流：输入先被树遍历置换，再编码，随后做相邻异或。逆向时应严格倒序撤销这些变换。末尾求和中的 `-28` 也不是额外密码学步骤，而是用未处理的首字节和编码长度将正确结果平衡为零。

最终 flag 为：

```text
miniLCTF{esrevER_gnir33nignE_Is_K1nd_0F_@rt}
```
