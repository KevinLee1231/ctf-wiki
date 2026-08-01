# Give Up

## 题目简述

附件脚本用一个固定字符串作为密钥，对整段明文进行循环异或。密文保存在 `you.txt`，题面与代码都暗示密钥被重复使用。目标是利用已知 flag 格式恢复循环密钥并解密。

## 解题过程

循环异或满足

$$
C_i=P_i\oplus K_{i\bmod L}.
$$

因此已知任意明文片段就会直接泄露同位置的密钥字节。题名和解密后的内容暗示 Rick Astley 的歌词，可以用开头 `We're no strangers to love` 做 crib；不依赖歌词时，也可以先以归一化汉明距离猜周期，再把各位置列转化为单字节 XOR，用英文频率评分。官方生成脚本进一步表明，循环密钥本身就是最终 flag。

最小恢复逻辑如下：

```python
cipher = bytes.fromhex(open("you.txt").read().strip())
known = b"We're no strangers to love\nYou know the rules and so do I"
keystream = bytes(c ^ p for c, p in zip(cipher, known))
print(keystream)  # 开头会显出循环的 flag 密钥
```

验证候选密钥时，把它循环异或整段密文；若输出形成连贯英文歌词，说明周期和密钥均正确。最终密钥也是答案：

```text
byuctf{n3v3r_g0nn4_r3us3_4_k3y}
```

## 方法总结

重复密钥 XOR 的安全性依赖密钥绝不复用且不能被已知明文对齐。固定格式、自然语言统计和重复周期都会迅速把它降为可解的多表替换问题；这也是流密码和一次一密中绝不能复用密钥流的原因。
