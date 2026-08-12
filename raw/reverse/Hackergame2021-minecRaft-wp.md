# minecRaft

## 题目简述

题面包装成 Minecraft 压力板小游戏，但三盏灯的真正判定完全位于前端混淆 JavaScript。`flag.js` 先旋转字符串数组隐藏常量，再给 `String.prototype` 增加加密函数。还原后可见校验使用 32 轮 TEA、固定 16 字节密钥和固定密文；因此主要障碍是 JavaScript 去混淆与算法识别，而不是 Minecraft 机制或服务端 Web 漏洞。

## 解题过程

### 去掉字符串数组混淆

先格式化 `flag.js`。文件开头的自执行函数不断执行：

```javascript
strings.push(strings.shift());
```

直到一串整数表达式等于常量。直接在浏览器控制台运行初始化部分，再打印索引函数的返回值，就能把 `_0x22517d(0x...)` 替换为真实字符串。关键校验最终化简为：

```javascript
function check(input) {
    const cipher = input.encrypt("1356853149054377");
    return cipher ===
        "6fbde674819a59bfa12092565b4ca2a7" +
        "a11dc670c678681daf4afb6704b82f0c";
}
```

加密函数中的常量 `0x9e3779b9`、两个 32 位字和 128 位密钥是 TEA 的典型特征。原代码每 4 个字符按小端序组成一个 word，但密文字符串把 word 按大端十六进制打印，解密时必须保持这两种字节序。

### 逆转 TEA

下面脚本完整复现 32 位溢出和原题字节序：

```python
DELTA = 0x9E3779B9
MASK = 0xFFFFFFFF

key_text = b"1356853149054377"
cipher = (
    "6fbde674819a59bfa12092565b4ca2a7"
    "a11dc670c678681daf4afb6704b82f0c"
)

key = [
    int.from_bytes(key_text[i:i + 4], "little")
    for i in range(0, 16, 4)
]

def decrypt_block(y, z):
    total = (DELTA * 32) & MASK
    for _ in range(32):
        z = (z - ((((y << 4) ^ (y >> 5)) + y) ^
                  (total + key[(total >> 11) & 3]))) & MASK
        total = (total - DELTA) & MASK
        y = (y - ((((z << 4) ^ (z >> 5)) + z) ^
                  (total + key[total & 3]))) & MASK
    return y, z

plain = bytearray()
for offset in range(0, len(cipher), 16):
    y = int(cipher[offset:offset + 8], 16)
    z = int(cipher[offset + 8:offset + 16], 16)
    y, z = decrypt_block(y, z)
    plain += y.to_bytes(4, "little")
    plain += z.to_bytes(4, "little")

print(plain.decode())
```

输出最短输入序列：

```text
McWebRE_inMlnCrA1t_3a5y_1cIuop9i
```

按题目规则加上外层即可得到：

```text
flag{McWebRE_inMlnCrA1t_3a5y_1cIuop9i}
```

## 方法总结

- 核心技巧：先执行或静态还原字符串数组索引，再通过 `0x9e3779b9` 和轮结构识别 TEA，按原字节序解密固定密文。
- 识别信号：静态网页、混淆 JS 中存在固定密钥/密文、32 轮双 word 运算与 TEA delta。
- 复用要点：Web 外壳不决定分类；前端秘密终究可读取。逆向 JS 位运算时要显式模拟 32 位溢出，并区分字符打包与十六进制输出的端序。
