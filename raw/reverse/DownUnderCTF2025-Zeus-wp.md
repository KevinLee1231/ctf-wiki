# Zeus

## 题目简述

附件是一个原生可执行程序。程序不会直接输出 flag，而是检查命令行参数：第一个参数必须是固定选项 `-invocation`，第二个参数必须与题面中的整段祷词完全一致。条件满足后，程序用循环异或解密内置的 51 字节密文。

## 解题过程

### 关键观察

在反编译结果的 `main` 函数中可以看到两次字符串比较，以及成功分支使用的密钥：

```c
const char *expected_phrase =
    "To Zeus Maimaktes, Zeus who comes when the north wind blows, "
    "we offer our praise, we make you welcome!";
const char *key = "Maimaktes1337";

if (argc == 3 &&
    strcmp(argv[1], "-invocation") == 0 &&
    strcmp(argv[2], expected_phrase) == 0) {
    xor(decrypted_flag, key);
    printf("His reply: %s\n", decrypted_flag);
}
```

因此最直接的解法是把完整短语作为一个带引号的参数传入：

```bash
./zeus -invocation "To Zeus Maimaktes, Zeus who comes when the north wind blows, we offer our praise, we make you welcome!"
```

也可以静态提取 `encrypted_flag` 与 `Maimaktes1337`，按源码中的规则恢复明文。第 $i$ 个字节满足：

$$
p_i = c_i \oplus k_{i \bmod 13},\qquad 0 \le i < 51.
$$

这两条路径互相验证：正确参数会触发程序自身的解密逻辑，离线异或则应得到相同结果。

## 方法总结

- 核心技巧：定位 `main` 中的参数比较和成功分支，再复现循环异或。
- 识别信号：程序提示“北风沉默”、题面给出一段异常具体的祷词，说明这段文本很可能直接参与参数校验。
- 复用要点：命令行中的整句必须作为单个参数传入；静态恢复时要使用真实密文长度，并按密钥长度循环，不能把结尾当作普通 C 字符串随意截断。
