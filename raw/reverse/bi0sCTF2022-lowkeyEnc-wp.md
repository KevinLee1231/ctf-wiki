# bi0sCTF 2022 lowkeyEnc Writeup

## 题目简述

lowkeyEnc 把 flag 连续执行两次 AES-256-CBC 加密，再将每个密文字节与其下标异或，最后把结果写入一张 $100\times100$ PNG 的最后一行。题目的漏洞不在 AES，而在 Go 标准库伪随机数生成器的使用：出题时的 Go 运行时中，全局 `math/rand` 未显式播种时使用固定状态，因此 32 字节密钥和 16 字节 IV 可以完全重放。

![几乎全黑的 100×100 载体图，最后一行前 96 个灰度像素依次编码异或后的双层 AES-CBC 密文](bi0sCTF2022-lowkeyEnc-wp/encrypted-pixel-row.png)

这张图的像素位置本身就是密文格式的一部分，具有独立视觉证据，因而保留。图中不是白色背景：源码虽然写着“fill it with white”，却没有真正填充，RGBA 缓冲区的默认值使其看起来几乎全黑。

## 解题过程

### 还原加密链

源码的有效变换可以写成：

$$
C_1=\operatorname{CBCEnc}_{K,IV}(\operatorname{PKCS7}(M)),
$$

$$
C_2=\operatorname{CBCEnc}_{K,IV}(\operatorname{PKCS7}(C_1)),
$$

$$
P_i=C_{2,i}\oplus(i\bmod256).
$$

`byteArrayToImage` 把 $P_i$ 同时写入像素 $(i,99)$ 的 R、G、B 三个通道。附件中的密文长度为 96 字节，所以只读取最后一行的前 96 个像素；其余像素是初始零值，不能拼进 CBC 密文。

### 重放密钥与 IV

加密程序依次调用：

```go
key := make([]byte, 32)
iv := make([]byte, 16)
rand.Read(key)
rand.Read(iv)
```

在题目使用的旧版 Go 中，包级 `math/rand` 默认等价于以 1 播种，所以求解端以相同调用顺序读取 32 字节和 16 字节，就得到完全相同的 $K$ 与 $IV$。

这一行为具有版本差异：Go 1.20 起默认自动播种，更新版本中 `rand.Seed` 的语义又有调整。最稳妥的复现方式是使用题目时代的 Go 工具链；若使用兼容版本，则在第一次 `rand.Read` 前显式执行 `rand.Seed(1)`。不能把“当前所有 Go 版本都固定输出同一序列”写成题目结论。

### 提取像素并逆向两层加密

官方求解器的核心流程如下：

```go
rand.Seed(1) // 使用与题目兼容的 Go 版本

key := make([]byte, 32)
iv := make([]byte, 16)
rand.Read(key)
rand.Read(iv)

f, _ := os.Open("enc.png")
img, _ := png.Decode(f)

data := make([]byte, 96)
for x := 0; x < 96; x++ {
    r, _, _, _ := img.At(x, 99).RGBA()
    data[x] = byte(r >> 8) ^ byte(x)
}

cbc := CBC256{Key: key, IV: iv}
firstLayer := cbc.DecryptByCBC(data)       // 同时移除外层 PKCS#7
plain := cbc.DecryptByCBC(firstLayer)      // 再移除内层 PKCS#7
fmt.Println(string(plain))
```

其中 `DecryptByCBC` 应先检查输入长度为 16 的倍数，并验证末字节指示的 PKCS#7 填充值，而不是无条件切片。按正确顺序先撤销下标异或，再解密并去填充两次，得到：

```text
bi0sCTF{Th3_fl4gs_l0wk3y_3ncryp7i0n_b3li3s_i7s_f0rmid4bl3_c1ph3r}
```

## 方法总结

本题需要分别识别三层表示：PNG 只是密文字节的空间载体，下标异或是可逆混淆，真正的分组密码则因密钥和 IV 来自可预测 PRNG 而失效。AES-CBC 本身没有被破解；安全边界在随机数生成处就已经丢失。

复现这类依赖标准库历史行为的题目时，应记录语言版本和默认播种语义。图片也不能仅凭“看起来全黑”就删除：最后一行的位置、通道和有效长度都是源码无法被一行 flag 文本取代的载体证据。
