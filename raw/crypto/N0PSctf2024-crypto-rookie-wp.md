# Crypto Rookie

## 题目简述

题目给出字符串：

```text
STsabeaoeoeieAlgsetrhnCoimmFItTAk
```

官方材料用手绘折线路径说明密文来自三栏栅栏密码（Rail Fence Cipher）。这张图表达的全部信息都可以用文本精确重建，因此不再保留代码式截图。

三栏栅栏的字符位置按 `0, 1, 2, 1, 0, ...` 往返变化。加密时先沿折线逐字符写入，再依次连接第 0、1、2 栏；解密则需要先还原每栏占据的位置，再沿原路径读取。

## 解题过程

密文长度为 34。先生成同长度的栏号序列，并统计三栏分别需要填入多少字符：

```text
栏号：0 1 2 1 0 1 2 1 0 ...
第 0 栏：STsabeaoe
第 1 栏：oeieAlgsetrhnCoi
第 2 栏：mmFItTAk
```

下面的脚本完成通用三栏栅栏解密：

```python
def rail_fence_decrypt(ciphertext: str, rail_count: int = 3) -> str:
    rail = 0
    direction = 1
    pattern = []

    for _ in ciphertext:
        pattern.append(rail)
        if rail == 0:
            direction = 1
        elif rail == rail_count - 1:
            direction = -1
        rail += direction

    counts = [pattern.count(index) for index in range(rail_count)]
    rails = []
    offset = 0
    for count in counts:
        rails.append(list(ciphertext[offset:offset + count]))
        offset += count

    positions = [0] * rail_count
    plaintext = []
    for index in pattern:
        plaintext.append(rails[index][positions[index]])
        positions[index] += 1
    return "".join(plaintext)


ciphertext = "STsabeaoeoeieAlgsetrhnCoimmFItTAk"
print(rail_fence_decrypt(ciphertext))
```

输出为：

```text
SomeTimesAFlagIsbetterThanACookie
```

按题目要求补上 flag 外壳：

```text
N0PS{SomeTimesAFlagIsbetterThanACookie}
```

## 方法总结

- 核心技巧：识别三栏栅栏密码的折线路径，并按每栏实际长度切分密文。
- 识别信号：字符未发生替换，只是顺序异常；手绘材料中存在明显的上下往返折线。
- 复用要点：解密时不能把密文平均分成若干段，因为首尾位置会使各栏长度不同；应先生成完整栏号序列，再据此统计、回填并读取。
