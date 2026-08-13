# GreyCTF 2023 Crackme1

## 题目简述

题目把验证逻辑放在经过混淆的 JavaScript 与 WebGL 2.0 transform feedback 中。页面会把用户输入重复扩展到 1024 字节，GPU 端状态机据此迭代 `0x104` 轮；同一份扩展输入随后又作为 RC4 类流密码的密钥，用来解开固定的 24 字节密文。图片只负责显示成功或失败界面，不参与验证，因此无需把界面截图保留到题解中。

## 解题过程

先还原字符串表，定位提交函数。与 flag 直接相关的末尾逻辑可以整理为：

```javascript
while (input.length < 0x400) input += input;
input = input.substring(0, 0x400);

const cipher = [
  0xc3, 0xb8, 0xb3, 0x42, 0xb6, 0xc2, 0x1c, 0xa4,
  0xce, 0x45, 0x06, 0x3b, 0x1f, 0x1c, 0x66, 0xb1,
  0x6c, 0x9a, 0x36, 0xe5, 0x14, 0xbf, 0x18, 0x6e
];
const stream = rc4Like(input, 24);
const plain = cipher.map((x, i) => x ^ stream[i]);
```

难点在于找出能让 WebGL 状态机进入成功纹理的原始短输入。顶点着色器的核心递推为

```glsl
f = (a * d + b + c * e) * step(0.0f, -abs(s.z));
```

每轮输出经取整和模 256 后成为下一轮状态，同时决定从三张常量表中选哪一行。把 transform feedback 的读回、四分量状态更新和输入索引翻译成普通整数运算后，逐轮倒推出使成功位成立的周期密钥为：

```text
REDchickenPIE
```

程序实际使用的是该字符串反复拼接并截断到 1024 字节的结果。将其送入页面中的 RC4 类 KSA/PRGA，再与固定密文异或即可得到：

```text
grey{y0u_h4d_fun?_e4a3d}
```

## 方法总结

这题的决定性障碍不是 Web 页面本身，而是 GPU transform feedback 隐藏的状态机。处理这类题时，应先把着色器输入、输出缓冲区和每轮读回关系恢复为普通代码，再分析其约束；渲染图片通常只是结果展示。找到短周期密钥后，后半段就是标准 RC4 状态初始化、伪随机字节生成与异或解密。
