# Baby's First RE

## 题目简述

附件 `baby` 是一个 64 bit ELF。程序本身没有复杂校验逻辑，题目重点是最基本的静态字符串检查。

## 解题过程

使用 `strings` 可以找到一段明显比普通符号长、且字符集符合 Base64 的文本：

```text
VU1EQ1RGLXsxX1NlZV9QcjMzdHlfU3RyaW5nel9ldmVyeXdoZXJlXzFfZ299
```

直接解码：

```python
import base64

encoded = "VU1EQ1RGLXsxX1NlZV9QcjMzdHlfU3RyaW5nel9ldmVyeXdoZXJlXzFfZ299"
print(base64.b64decode(encoded).decode())
```

得到：

```text
UMDCTF-{1_See_Pr33ty_Stringz_everywhere_1_go}
```

其 SHA-256 与 README 中的 `dedc4a8f16c2c1fab892d8be7bf5f8eeefc360dc250dc4f4dd968cc9833ddd9f` 一致。

## 方法总结

逆向的第一步应当是低成本静态侦察：文件类型、保护、导入和可打印字符串。本题的长 Base64 串已经直接携带答案，不需要先进入反编译器。解码后仍用官方摘要校验大小写和数字，避免人工抄写出错。
