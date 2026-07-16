# ezshell_PLUS

## 题目简述

登录受限 Shell 后，用户目录的 `challenge` 中包含 100 个随机数据文件、一个真正的加密 flag 文件、目标文件的 SHA-256 值以及 `decrypt.sh`。生成脚本表明 flag 使用 OpenSSL AES-256-CBC、salt 与 PBKDF2 加密；解密脚本能够读取 `/etc/secret_key.backup`，因此不需要破解 AES 密钥，关键是从垃圾文件中找出正确密文。

## 解题过程

进入 `challenge` 后读取 `hash_value`。该值是在环境初始化时对真正的加密 flag 文件计算的 SHA-256，因此可以对 `files` 目录逐个校验并筛出唯一匹配项：

```bash
cd ~/challenge
target=$(cat hash_value)
sha256sum files/* | grep "$target"
```

得到匹配文件路径后，将其交给题目自带脚本：

```bash
./decrypt.sh files/<matched-file>.dat
```

`decrypt.sh` 的关键操作是先 Base64 解码，再以与加密端相同的参数调用 OpenSSL：

```bash
base64 -d "$1" | openssl enc -d -aes-256-cbc -pbkdf2 \
  -pass file:/etc/secret_key.backup
```

备份密钥文件属于 `root:welcome` 且对组可读，当前用户可以通过脚本正常使用它。最终输出为 `0xGame{Welc0me_to_H@ckers_w0r1d}`。

## 方法总结

- 核心技巧：用已知 SHA-256 指纹从大量同类文件中定位目标，再复用题目提供的合法解密链。
- 识别信号：垃圾文件目录、独立哈希值、可执行解密脚本和受控密钥文件同时出现。
- 复用要点：先读脚本与权限，判断是否已有可利用的解密能力；摘要用于定位文件，不是要被“解密”的密文。
