# adorable encrypted animal

## 题目简述

题目给出 Apple Archive Encryption（AEA）格式的 `cat.png.aea` 与 `flag.txt.aea`。生成程序把固定 32 字节主密钥用于加密已知猫图，再从猫 archive 的两个位置取出 $K_1$、$K_2$，将原位置以 32 个 `Z` 覆盖；flag archive 的实际 key 是 $K_1\oplus K_2$。因此并非暴力破解 AES，而是还原 AEA 的 HKDF、CTR、HMAC 派生链，从已知明文 archive 恢复两个被抹去的材料。

猫图已人工查看，只是普通猫照片；其视觉外观不提供额外线索，解法使用的是解密后完整文件的 SHA-256，故不复制图片资源。

## 解题过程

### 解开已知的 cat archive

生成器中硬编码的 `KEY` 以 `hex:` 参数传给系统 `aea encrypt`。官方 solver 先读取 archive 的 salt，按 AEA 的 master-key context `AEA_AMK\x01\0\0\0` 作 HKDF-SHA256，得到 master key。随后：

1. 以 context `AEA_CK\0\0\0\0` 与 `AEA_CHEK` 派生 cluster-header 的 key/IV，用 AES-CTR 解密 $0x2800$ 字节 header；
2. 从 40 字节 segment descriptor 取 `raw_size`；
3. 以同一 intermediate key 和 `AEA_SK\0\0\0\0` 派生 data 的 MAC key、CTR key、IV，解密 cat segment。

这一步所恢复的是完整猫 PNG 字节，而非图像识别结果。

### 恢复 $K_1$、$K_2$ 并解 flag

被覆盖前的两个 32 字节值本来在 cat archive 的 header/data 认证材料中。官方过程用解密 header 后连续的 CTR keystream 加密 `b"x"*8 + sha256(cat_png).digest()`，去掉前 8 字节得到 $K_1$；又按 AEA data MAC 规则计算：

$$K_2=\operatorname{HMAC}_{\text{SHA-256}}(\text{mac\_key},\; \text{ciphertext}\parallel 0^8).$$

于是 flag archive 的 key 为 $K_1\oplus K_2$。对其重复同样的 AEA header/segment 解密即可恢复明文。原 C 程序中的偏移 `0xa4` 与 `0x28bc` 解释了为何恰好需要这两份材料。

### 验证

题目源码与配置中的结果为 `DUCTF{h0pe_y0u_enjoy3d_th3_fr33_cat_p1c_:)}`。本文未执行 Apple 的 `aea` 工具或解密脚本；算法顺序来自生成 C 代码和官方 solver 的静态对照。

## 方法总结

- 核心技巧：已知明文加密容器可以成为恢复被抹去的认证/keystream材料的依据，前提是精确复现容器的 KDF、MAC 与 CTR 状态。
- 识别信号：程序先加密一个已知文件、再改写 archive 内少数块，并以这些块的 XOR 加密第二文件时，应分析文件格式而非把问题笼统当作 AES 破解。
- 复用要点：AEA 的 context 字符串、segment 偏移、CTR 流位置和零填充都影响派生结果；猫图仅作为字节级 known plaintext，图片本身无需保存。
