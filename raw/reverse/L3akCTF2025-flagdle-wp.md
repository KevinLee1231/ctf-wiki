# L3akCTF 2025 flagdle Writeup

## 题目简述

`flagdle` 是一个 Go 编写的 Wordle 游戏。正常流程要求最后一次猜中目标词，同时此前每一行的灰、黄、绿状态都与配置中的哈希一致；特定格子的字符再与密钥异或，才会组成 flag。

题目同时提供了 `flagdle.dat`。配置的加密密钥、目标词、行哈希、字典和 flag 校验值都在该文件中，因此无需人工完成随机抽取的游戏。决定性障碍是还原配置格式和枚举满足行状态的单词，本文按 Reverse 归档。

## 解题过程

### 解开 Protobuf 配置

程序的 Protobuf 定义表明，外层 `EncryptedSettings` 包含加密模式、密钥和 `gameSettings` 密文；内层 `Config` 保存全部 `Game`、单词长度、字典和 flag 的 MD5。每个 `Game` 又保存：

```text
target          目标单词
rowHashes       各行 Wordle 状态的 MD5
flagKey         每个 flag 字节的异或密钥
flagSelectors   从哪一行、哪一列取字符
```

附件采用 AES 模式。密钥与密文本身放在同一个外层消息中，CBC 的 IV 固定为 16 个零字节。按 PKCS#7 去除填充后即可再次用 Protobuf 解析：

```go
var outer pb.EncryptedSettings
proto.Unmarshal(data, &outer)

block, _ := aes.NewCipher(outer.Key)
plain := make([]byte, len(outer.GameSettings))
iv := make([]byte, aes.BlockSize)
cipher.NewCBCDecrypter(block, iv).CryptBlocks(plain, outer.GameSettings)

pad := int(plain[len(plain)-1])
plain = plain[:len(plain)-pad]

var cfg pb.Config
proto.Unmarshal(plain, &cfg)
```

原程序会从网络下载这个文件，但仓库已经给出同一份 `flagdle.dat`，本地文件足以完整求解，不需要保留易失的比赛服务地址。

### 反推每一行可能的猜词

行哈希并不包含字母，只计算 5 个格子状态的 MD5：

```text
Grey = 1, Yellow = 2, Green = 3
rowHash = MD5(status[0] || ... || status[4])
```

每局的目标词是已知的，所以可以遍历配置中的字典，对每个候选词执行标准 Wordle 判定：先标记位置正确的绿色，再用目标词尚未消费的字母标记黄色，剩余为灰色。状态数组的 MD5 与目标 `rowHash` 相等时，该候选词就是这一行的可能输入。

重复字母必须按照上述“两遍判定”处理。若只判断字母是否出现在目标词中，会重复消费同一个目标字母，从而产生错误的黄色格子。

### 逐局收缩 flag 字节

设某个 selector 指向第 $r$ 行第 $c$ 列，对这一行所有候选词取字符并与对应的 `flagKey` 字节异或：

```go
for _, word := range rowCandidates[r] {
    b := word[c] ^ game.FlagKey[i]
    if b >= 32 && b <= 127 {
        choices[b] = true
    }
}
```

同一 flag 位置会在不同游戏中重复出现。为每个位置维护一个初始为所有可打印 ASCII 的集合，并对 1000 局结果连续求交集。到第 14 局时，每个位置已经只剩一个字符；最后再用配置中的 `flagHash` 验证完整结果：

```text
L3AK{m4yb3_th3_r341_w0rd_w45_th3_fr13nd5_w3_m4d3_al0ng_th3_w4y}
```

该结果的 MD5 与内层配置完全一致。

## 方法总结

本题看似要求操纵 Wordle 游戏，实际配置已经泄露了解题所需的全部约束。AES 只起到包装作用，因为密钥和固定 IV 都能从客户端逻辑中直接获得；真正的工作是把不可逆的行哈希转化为一个可枚举的小搜索空间。

当哈希输入只有有限状态组合时，不应尝试攻击 MD5 本身。利用已知目标词和内置字典枚举候选行，再让大量独立游戏对 flag 字节集合求交，能够把每局的歧义快速消掉。最终的整串哈希则用于确认收缩结果，而不是用于直接反演 flag。
