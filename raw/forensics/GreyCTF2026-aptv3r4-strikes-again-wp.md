# APTV3R4_STRIKES_AGAIN

## 题目简述

题目给出了一份 `artifact.pcap`，背景是攻击者已经取得文件保险库的访问令牌和 SMB 传输中的解密密钥，却还缺少内存中打开的 `flag.enc`。保险库前端说明下载请求需要 67 字符 token；后端在请求 `mem_dump.dmp` 时会重定向到内存转储压缩包。真正的难点不是 Web 绕过，而是从 PCAP 中恢复 token、按 SMB 3/NTLMv2 会话协商导出流量加密密钥，再把内存转储与加密 keyfile 组合起来。

`flag.enc` 是 OpenSSL `enc` 的盐化 AES-256-CBC 数据：开头为 `Salted__`，使用 PBKDF2-HMAC-SHA256 派生 32 字节 AES 密钥和 16 字节 IV，迭代次数为 10000。SMB payload 中的 keyfile 是该 OpenSSL 数据的口令。

## 解题过程

### 取得内存载体

先在 PCAP 原始字节中搜索长度恰为 67 的字母数字 token，再从携带该 token 的 HTTP 请求取得保险库主机名。以该 token 请求 `download=mem_dump.dmp` 会取得 ZIP 形式的内存转储；不需要把临时服务地址写入长期资料。

转储中不是完整文件系统，而是包含下列边界标记的内存残留：

```text
BEGIN_REAL_ARTIFACT_flag.enc
Salted__...
END_REAL_ARTIFACT
```

在两个标记之间截取 `Salted__` 起始的字节串，并去除为满足 CBC 分组长度而附带的空白或 NUL，得到 `flag.enc`。题目官方 solver 会在未找到标记时才退回为挑选 64 字节 `Salted__` 候选；正常路径应优先使用完整标记，避免误取内存中的其他盐化数据。

### 解开 SMB3 读取响应

PCAP 中先有 SMB2 NEGOTIATE 与 NTLMSSP type 1、2、3 消息，随后是 `\xfdSMB` SMB3 transform packet。已知题目配置的 SMB 口令为 `mypassword` 时，按下列顺序恢复两个方向的会话加密键：

1. 从 type 2 取服务器 challenge，从 type 3 取 domain、user、NT response 与 encrypted session key。
2. 计算 `NTOWFv2 = HMAC-MD5(NT_hash(password), UTF-16LE(upper(user) || domain))`，验证 NTLMv2 proof，确保没有把错误会话当成目标会话。
3. 以 proof 求 session base key；若 type 3 带 encrypted session key，则用 ARC4 解开 exported session key。
4. 将 negotiate request/response 和三条 session-setup 原文依序滚动做 SHA-512，得到 pre-auth integrity hash；再以 SMB counter-mode KDF 的 `SMBC2SCipherKey`、`SMBS2CCipherKey` 标签派生 client-to-server 与 server-to-client 的 128 位 key。

逐个解密 transform packet，并只保留成功的 SMB2 `READ` response 的 data buffer。题目流量的 algorithm 字段看似 CCM，但实测 nonce 为 12 字节的 AES-GCM；因此应先按声明尝试 CCM，认证失败后再尝试 GCM。随后在拼接的 READ buffer 中取十六进制 keyfile。

官方 solver 可把以上两条证据链串起来；将内存 ZIP 显式传入可避免依赖已关闭的比赛服务：

```bash
python solve_aptv3r4_strikes_again.py \
  --challenge-dir /path/to/APTV3R4_STRIKES_AGAIN \
  --memdump-zip /path/to/mem_dump.zip \
  --outdir solve_out
```

### 解密与验证

恢复出的 keyfile 与 carved `flag.enc` 可以直接交给 OpenSSL；`-pbkdf2` 与原始加密参数一致。

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -salt \
  -in solve_out/flag.enc \
  -pass file:solve_out/keyfile.txt
```

明文的 flag 格式校验通过，结果为 `grey{7r1v14l_70_f0ll0w_7h3_5mb3_7r41l}`。

## 方法总结

- 核心技巧：把 HTTP token、内存 carving 和 SMB3 会话解密作为同一条证据链，而不是把 SMB 加密流量当作不可用噪声。
- 识别信号：PCAP 同时出现 NTLM type 1/2/3 和 `\xfdSMB` 时，应先确认是否有已知口令或可验证的 NTLMv2 proof，再推导 SMB session key。
- 复用要点：内存中提取 OpenSSL `Salted__` 数据时应优先依赖上下文标记；处理 SMB transform 时以 AEAD 认证成功和内层 `\xfeSMB` 头为准，不能盲信算法字段。
