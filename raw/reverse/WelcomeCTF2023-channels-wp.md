# Go Big or Go Hash

## 题目简述

题目给出一段 URL-safe Base64 字符串。Go 程序把 Flag 依次经过 11 层变换，每层先按指定数量切块，再按块序号循环使用 Add、Rotate、Xor、ReverseBits、Shuffle 五种函数。层大小为：

```text
1, 2, 4, 8, 16, 32, 16, 8, 4, 2, 1
```

各层使用 goroutine 并发处理，但通过 `Index` 把结果放回原位置，因此并发完成顺序不会改变密文结构。

## 解题过程

先做 URL-safe Base64 解码，然后逆序遍历 11 层。每层仍按对应大小重新切块，并根据块序号选择逆函数：

- `Add`：使用由块长度播种的 Go PRNG，并利用已恢复的前一个明文字节撤销扩散加法；
- `Rotate`：用相同种子重算旋转量，再反向旋转；
- `Xor`：XOR 自逆，但扩散项必须使用已经恢复的前一个明文字节；
- `ReverseBits`：8 位反转执行两次即还原；
- `Shuffle`：用相同种子重建排列，再把每个密文字节放回原索引。

核心逆向框架如下：

```go
func decrypt(acc string) string {
    raw, _ := base64.URLEncoding.DecodeString(acc)
    acc = string(raw)

    for layer := len(layerSizes) - 1; layer >= 0; layer-- {
        parts := splitStringIntoParts(acc, layerSizes[layer])
        out := make([]string, len(parts))
        for i, part := range parts {
            out[i] = inverseFunctions[i%len(inverseFunctions)](part)
        }
        acc = strings.Join(out, "")
    }
    return acc
}
```

对题目字符串：

```text
MsH-J7hjdv7XDBpOM67U8QkhoCmBFn0VkZYT9jm5cft2
```

运行官方逆变换程序得到：

```text
greyhats{w0w_n0w_try_w1th0ut_srC}
```

## 方法总结

- 核心技巧：按相反层序执行确定性分块变换的逆函数，并复现 Go PRNG 的播种方式。
- 识别信号：多层可逆变换、每层固定分块数、随机操作实际由块长度固定播种。
- 复用要点：逆扩散必须使用已经恢复的明文字节；并发代码要先确认结果是否通过索引重新排序，不能把调度顺序误当作加密随机性。
