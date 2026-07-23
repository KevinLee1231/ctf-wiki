# Hall's Harmony

## 题目简述

附件是一段由固定时长单音组成的音频。仓库中的生成与解密代码表明，七种音高分别对应字母 `a` 至 `g`；每三个音符组成一个三位七进制数，再转换为一个明文字节。

## 解题过程

音频每个音符持续约 0.5 秒。按时间切片后测出主频，把七档音高依次映射为 `a` 至 `g`，可恢复仓库中对应的音符串：

```text
bfbbeabcfbcebfabdaagdcdecabcbdbcbccdccdbaacabbadcbdbgecabccccdcccacceaggcdg
```

令 `a=0`、`b=1`，直到 `g=6`。每三个字符 $(x_0,x_1,x_2)$ 对应：

$$
m = 49x_0 + 7x_1 + x_2
$$

实现如下：

```python
notes = "bfbbeabcfbcebfabdaagdcdecabcbdbcbccdccdbaacabbadcbdbgecabccccdcccacceaggcdg"
plain = bytearray()

for i in range(0, len(notes), 3):
    digits = [ord(ch) - ord("a") for ch in notes[i:i + 3]]
    plain.append(digits[0] * 49 + digits[1] * 7 + digits[2])

print(plain.decode())
```

结果为：

```text
UMDCTF-{cl@ss1c4l_crypt0}
```

## 方法总结

音频在本题中只是七进制数字的物理载体。先按固定符号时长恢复离散音高，再按三位一组解释为七进制，比直接尝试语音识别或把频率当 ASCII 更符合生成机制。
