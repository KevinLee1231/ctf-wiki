# Let's Catch Up

## 题目简述

网页让用户输入“加密密钥”，前端先用一个看似固定的 AES-256 密钥加密邮件，再用用户输入尝试解密并比较明文。真正的 AES 实现藏在经过自定义混淆的 `a.js` 中，而且库在密钥扩展后覆盖了轮密钥，所以页面上显示的垃圾密钥并不决定密文。

## 解题过程

`a.js` 顶部留下的注释暗示作者修改了某个公开库；文件中的
`0xc66363a5` 等 AES T-table 常量可将范围缩小到 `aes-js`。对照原库结构，定位构造函数中的密钥扩展和 CBC 包装器，可以发现扩展后的加密轮密钥 `Ke` 被一组常量替换。

数组共有 15 轮，每轮四个 32 位字，对应 AES-256 的轮密钥布局。AES 密钥扩展的最前八个字就是原始 $256$ 位密钥，因此无需逆完整轮函数，只需读取：

```text
Ke[0][0..3]
Ke[1][0..3]
```

每个 JavaScript 数值虽然可能显示为负数，但位运算仍按 32 位二补码处理。按大端顺序拆成四个字节：

```javascript
const byte0 = x => (x >>> 24) & 0xff;
const byte1 = x => (x >>> 16) & 0xff;
const byte2 = x => (x >>> 8) & 0xff;
const byte3 = x => x & 0xff;

const key = [];
for (let i = 0; i < 8; i++) {
    const word = Ke[Math.floor(i / 4)][i % 4];
    key.push(byte0(word), byte1(word), byte2(word), byte3(word));
}
console.log(String.fromCharCode(...key));
```

恢复出的 32 字节密钥本身就是 flag：

```text
UMDCTF{y4y_1_made_some_new_k3y5}
```

输入该字符串后，第二个 CBC 实例生成的轮密钥与硬编码数组一致，页面成功把密文解回原邮件并显示发送成功。

## 方法总结

- 核心技巧：用 AES 固定 T-table 常量识别被改名、混淆的第三方实现，再与上游库做结构对照。
- 前端传入的可见密钥是障眼法；真正决定结果的是密钥扩展后被覆盖的 `Ke`。
- 对 AES-256，最初八个 32 位轮密钥字就是原始 32 字节密钥。
- JavaScript 位运算有符号显示不影响取字节，但应使用无符号右移 `>>>` 避免符号扩展干扰理解。
