# 特工 luo：闻风而动

## 题目简述

附件包含无线抓包 `catflag.cap` 和二次加密文件 `flag.fromserver`。完整链条是：逆向易语言 keygen 缩小 Wi-Fi 密码候选空间，利用 WPA2 四次握手验证候选密码，再按保存流程的相反顺序撤销 RC4 与 DES。

## 解题过程

### 1. 从 keygen 生成候选密码

客户端连接服务器时使用由 Wi-Fi 密码派生出的口令。keygen 的核心只是逐字符异或；根据异或常量和输出字符范围，可以区分某一位应为数字还是字母，不必对完整可打印字符集做盲目爆破。将每一位的允许集合组合成候选字典。

### 2. 用握手包验证候选

抓包中包含 WPA2 握手。先转为 Hashcat 的 22000 格式，再使用 keygen 产生的候选字典：

```bash
hcxpcapngtool -o catflag.22000 catflag.cap
hashcat -m 22000 catflag.22000 candidates.txt
```

命中的 Wi-Fi 密码为：

```text
4g3n71u0
```

### 3. 逆序解开文件

服务端发送的是用 Wi-Fi 密码做 DES 加密的数据；客户端保存时又以文件名 `flag.fromserver` 为口令做了一层 RC4。因此解密顺序必须反过来：

```text
plain = DES_decrypt(
    RC4_decrypt(read("flag.fromserver"), "flag.fromserver"),
    "4g3n71u0"
)
```

用附件客户端中的同一套易语言“解密数据”实现，可写成：

```text
输出调试文本(
    到文本(
        解密数据(
            解密数据(读取文件("flag.fromserver"), "flag.fromserver", #RC4加密),
            "4g3n71u0",
            #DES加密
        )
    )
)
```

这里应复用客户端所用 DES 模式、填充方式和 RC4 实现，不能只凭算法名称换成任意库的默认参数。最终得到：

```text
moectf{h4cker_Lu0Q1@n_is_trying_to_0p3n_your_r3g15t3r3d_res1denc3}
```

## 方法总结

这题不是单独的 Wi-Fi 跑字典：keygen 逆向负责把候选空间压缩到可行规模，握手包只负责离线验证，最终还要按客户端数据流逆序解开 RC4 和 DES。文件名既是载荷名也是外层口令，是最容易漏掉的关联。
