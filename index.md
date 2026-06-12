# CTF Wiki Index

这是 `ctf-*` skills 的外接技巧知识图谱。它不是默认上下文；只有当 `SKILL.md` 的直链页面不足、需要跨方向关联、详细工具说明或原始资料出处时才读取。

- Knowledge base path: `D:/文档/markdown文件/ctf-wiki`
- Structure: flat graph
- Updated: 2026-06-11

## 查询顺序

```text
已激活 ctf-* skill
  -> SKILL.md
  -> SKILL.md 中的 wiki 直链 / <direction>-tooling.md
  -> 如不足，再查本 index
  -> wiki/<family|technique|tooling>.md
  -> 必要时回查 raw/<direction>/
```

## 页面类型

`wiki/` 只使用三类页面：

| Type | 作用 | raw 关系 |
|---|---|---|
| `family` | 分流页、技巧族入口、跨技巧 map。负责连接多个 technique，并说明首轮判断、变体 pivot 和失败后转向。 | 可以挂 raw，尤其是承载大量 WP case 的方向入口页。 |
| `technique` | 具体可复用技巧、攻击模式、恢复路径或判断模型。负责适用场景、识别信号、最小证据、解法骨架和常见坑。 | 可以挂 raw，作为案例和证据来源。 |
| `tooling` | 工具入口、环境限制、稳定调用方式和常见失败状态。 | 只挂工具资料或与工具调用直接相关的 raw。 |

## Family Pages

| Family | 方向 | 用途 |
|---|---|---|
| [block-mode-misuse-family.md](wiki/block-mode-misuse-family.md) | crypto | 分组模式、MAC、oracle 和对称密码误用分流。 |
| [crypto-parameter-triage-family.md](wiki/crypto-parameter-triage-family.md) | crypto | RSA/ECC/格/PRNG/哈希/代数等 crypto 参数首轮分流。 |
| [sqli-filter-and-oracle-family.md](wiki/sqli-filter-and-oracle-family.md) | web | SQLi 过滤、盲注、二阶注入和 oracle 分流。 |
| [web-first-pass-triage-and-chain-patterns.md](wiki/web-first-pass-triage-and-chain-patterns.md) | web | Web 首轮业务流、解析差异和漏洞链分流。 |
| [auth-bypass-cookies-and-hidden-routes.md](wiki/auth-bypass-cookies-and-hidden-routes.md) | web | Cookie/session、隐藏路由、代理 ACL、OAuth 和访问控制分流。 |
| [php-lfi-ssti-ssrf-and-type-juggling.md](wiki/php-lfi-ssti-ssrf-and-type-juggling.md) | web | PHP/LFI/SSTI/SSRF/XXE/type juggling 等服务端解释层差异分流。 |
| [path-traversal-ssrf-upload-and-rsc.md](wiki/path-traversal-ssrf-upload-and-rsc.md) | web | 路径穿越、上传、SSRF、渲染器、RSC 和内部服务链分流。 |
| [ruby-php-upload-and-ssti-rce.md](wiki/ruby-php-upload-and-ssti-rce.md) | web | 语言 eval、模板、上传、反序列化和命令执行链分流。 |
| [sqli-upload-deser-and-command-rce.md](wiki/sqli-upload-deser-and-command-rce.md) | web | SQLi、上传、反序列化、命令包装器和文件读到 RCE 的链路分流。 |
| [known-cves-and-n-day-exploits.md](wiki/known-cves-and-n-day-exploits.md) | web | CVE/N-day 版本边界、最小 PoC 和后续漏洞链分流。 |
| [xss-dom-and-browser-tricks.md](wiki/xss-dom-and-browser-tricks.md) | web | XSS、DOM、admin bot、缓存/MIME、CSP 和浏览器外带分流。 |
| [pwn-first-pass-red-flags-and-protections.md](wiki/pwn-first-pass-red-flags-and-protections.md) | pwn | Pwn 首轮保护、漏洞族和最短利用路线分流。 |
| [oob-jit-parser-primitives-family.md](wiki/oob-jit-parser-primitives-family.md) | pwn | OOB/JIT/parser primitive 分流。 |
| [cross-primitive-escape-and-hybrid-exploit-map.md](wiki/cross-primitive-escape-and-hybrid-exploit-map.md) | pwn | 多 primitive、沙箱和混合利用路线 map。 |
| [reverse-first-pass-workflow-and-debugging.md](wiki/reverse-first-pass-workflow-and-debugging.md) | reverse | 逆向首轮载体、调试、dump 和比较点分流。 |
| [vm-obfuscation-transform-family.md](wiki/vm-obfuscation-transform-family.md) | reverse | VM、字节码、变换链和解释器类题分流。 |
| [cross-domain-forensics-technique-map.md](wiki/cross-domain-forensics-technique-map.md) | forensics | 跨 PCAP/磁盘/内存/媒体/容器取证的下一跳 map。 |
| [pcap-protocol-credential-recovery-family.md](wiki/pcap-protocol-credential-recovery-family.md) | forensics | PCAP、协议重组、凭据和 key 恢复分流。 |
| [misc-cross-category-triage-family.md](wiki/misc-cross-category-triage-family.md) | misc | 编码、游戏、shell、oracle、pyjail、轻量附件等 misc 边界题分流。 |
| [file-triage-archives-and-one-liners.md](wiki/file-triage-archives-and-one-liners.md) | misc | 文件首检、压缩包、一行式和轻量附件 triage。 |

## Tooling Pages

| Direction | Tooling |
|---|---|
| ai-ml | [ai-ml-tooling.md](wiki/ai-ml-tooling.md) |
| crypto | [crypto-tooling.md](wiki/crypto-tooling.md) |
| forensics | [forensics-tooling.md](wiki/forensics-tooling.md) |
| malware | [malware-tooling.md](wiki/malware-tooling.md) |
| misc | [misc-tooling.md](wiki/misc-tooling.md) |
| osint | [osint-tooling.md](wiki/osint-tooling.md) |
| pentest | [pentest-tooling.md](wiki/pentest-tooling.md) |
| pwn | [pwn-tooling.md](wiki/pwn-tooling.md) |
| reverse | [reverse-tooling.md](wiki/reverse-tooling.md) |
| reverse | [disassemblers-debuggers-and-basic-tools.md](wiki/disassemblers-debuggers-and-basic-tools.md) |
| reverse | [frida-angr-lldb-and-x64dbg.md](wiki/frida-angr-lldb-and-x64dbg.md) |
| reverse | [qiling-triton-pin-and-ldpreload.md](wiki/qiling-triton-pin-and-ldpreload.md) |
| web | [web-tooling.md](wiki/web-tooling.md) |

## Technique Pages

### AI / ML

- [adversarial-ml.md](wiki/adversarial-ml.md)
- [llm-attacks.md](wiki/llm-attacks.md)
- [ml-model-inference-extraction-and-weight-analysis.md](wiki/ml-model-inference-extraction-and-weight-analysis.md)

### Crypto / Blockchain

- [blockchain-smart-contract-exploitation.md](wiki/blockchain-smart-contract-exploitation.md)
- [classical-xor-and-substitution-ciphers.md](wiki/classical-xor-and-substitution-ciphers.md)
- [ecc-dlp-and-signature-attacks.md](wiki/ecc-dlp-and-signature-attacks.md)
- [exotic-secret-sharing-rabin-and-polynomials.md](wiki/exotic-secret-sharing-rabin-and-polynomials.md)
- [hash-protocol-and-oracle-attacks.md](wiki/hash-protocol-and-oracle-attacks.md)
- [homomorphic-and-exotic-algebra.md](wiki/homomorphic-and-exotic-algebra.md)
- [lattice-and-lwe.md](wiki/lattice-and-lwe.md)
- [lorenz-and-book-cipher-attacks.md](wiki/lorenz-and-book-cipher-attacks.md)
- [mt-lcg-and-seed-recovery.md](wiki/mt-lcg-and-seed-recovery.md)
- [number-theory-and-algebra-attacks.md](wiki/number-theory-and-algebra-attacks.md)
- [prng-z3-lcg-and-timing-attacks.md](wiki/prng-z3-lcg-and-timing-attacks.md)
- [rc4-lfsr-and-keystream-reuse.md](wiki/rc4-lfsr-and-keystream-reuse.md)
- [rsa-attacks.md](wiki/rsa-attacks.md)
- [rsa-specialized-structures-and-oracles.md](wiki/rsa-specialized-structures-and-oracles.md)
- [zkp-secret-sharing-and-proof-systems.md](wiki/zkp-secret-sharing-and-proof-systems.md)

### Web

- [artifact-trust-ssrf-to-node-require-rce.md](wiki/artifact-trust-ssrf-to-node-require-rce.md)
- [auth-edge-cases-and-protocol-bypasses.md](wiki/auth-edge-cases-and-protocol-bypasses.md)
- [auth-jwt.md](wiki/auth-jwt.md)
- [csp-xsleak-and-browser-exfiltration.md](wiki/csp-xsleak-and-browser-exfiltration.md)
- [game-state-websocket-and-wasm.md](wiki/game-state-websocket-and-wasm.md)
- [json-duplicate-key-hmac-parser-differential.md](wiki/json-duplicate-key-hmac-parser-differential.md)
- [node-and-prototype.md](wiki/node-and-prototype.md)
- [oauth-saml-cors-and-cicd.md](wiki/oauth-saml-cors-and-cicd.md)
- [parser-wrapper-and-legacy-ssrf-tricks.md](wiki/parser-wrapper-and-legacy-ssrf-tricks.md)
- [path-confusion-to-signed-internal-request-chain.md](wiki/path-confusion-to-signed-internal-request-chain.md)
- [php-java-python-deserialization.md](wiki/php-java-python-deserialization.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](wiki/polyglot-url-tricks-and-ssrf-leaks.md)
- [workflow-runner-internal-api-chain.md](wiki/workflow-runner-internal-api-chain.md)
- [xml-command-and-graphql-injection.md](wiki/xml-command-and-graphql-injection.md)

### Pwn

- [emulator-float-and-hash-exploits.md](wiki/emulator-float-and-hash-exploits.md)
- [format-string.md](wiki/format-string.md)
- [heap-fsop-file-structure-attacks.md](wiki/heap-fsop-file-structure-attacks.md)
- [heap-houses-unlink-and-tcache.md](wiki/heap-houses-unlink-and-tcache.md)
- [heap-uaf-tcache-and-custom-allocator.md](wiki/heap-uaf-tcache-and-custom-allocator.md)
- [interpreter-jit-canary-and-integer-exploits.md](wiki/interpreter-jit-canary-and-integer-exploits.md)
- [kaslr-kpti-smep-and-kernel-debugging.md](wiki/kaslr-kpti-smep-and-kernel-debugging.md)
- [kernel-uaf-race-and-slab-techniques.md](wiki/kernel-uaf-race-and-slab-techniques.md)
- [linux-kernel-exploit-basics.md](wiki/linux-kernel-exploit-basics.md)
- [overflow-basics.md](wiki/overflow-basics.md)
- [python-vm-and-proc-sandbox-escape.md](wiki/python-vm-and-proc-sandbox-escape.md)
- [race-condition-and-concurrency-exploits.md](wiki/race-condition-and-concurrency-exploits.md)
- [ret2csu-dynelf-and-shellcode.md](wiki/ret2csu-dynelf-and-shellcode.md)
- [runtime-protection-and-tls-exploits.md](wiki/runtime-protection-and-tls-exploits.md)
- [seccomp-ret2dlresolve-and-runtime-primitives.md](wiki/seccomp-ret2dlresolve-and-runtime-primitives.md)
- [stack-pivots-srop-and-seccomp-rop.md](wiki/stack-pivots-srop-and-seccomp-rop.md)
- [windows-arm-and-cross-platform-exploits.md](wiki/windows-arm-and-cross-platform-exploits.md)

### Reverse

- [android-games-hardware-and-runtime-platforms.md](wiki/android-games-hardware-and-runtime-platforms.md)
- [anti-analysis.md](wiki/anti-analysis.md)
- [compare-breakpoint-plaintext-recovery.md](wiki/compare-breakpoint-plaintext-recovery.md)
- [embedded-python-pyd-custom-aes.md](wiki/embedded-python-pyd-custom-aes.md)
- [font-shader-firmware-and-legacy-patterns.md](wiki/font-shader-firmware-and-legacy-patterns.md)
- [go-rust-jvm-and-cpp-reversing.md](wiki/go-rust-jvm-and-cpp-reversing.md)
- [hardware-isa-bootloader-and-kvm.md](wiki/hardware-isa-bootloader-and-kvm.md)
- [loader-vm-image-and-kernel-patterns.md](wiki/loader-vm-image-and-kernel-patterns.md)
- [mobile-firmware-kernel-and-game-re.md](wiki/mobile-firmware-kernel-and-game-re.md)
- [packers-deobfuscation-and-debug-automation.md](wiki/packers-deobfuscation-and-debug-automation.md)
- [python-bytecode-esolangs-and-uefi.md](wiki/python-bytecode-esolangs-and-uefi.md)
- [runtime-patching-oracles-and-tracing.md](wiki/runtime-patching-oracles-and-tracing.md)
- [self-decrypting-strings-and-lattice-patterns.md](wiki/self-decrypting-strings-and-lattice-patterns.md)
- [signal-trace-and-packed-anti-analysis.md](wiki/signal-trace-and-packed-anti-analysis.md)
- [signals-and-hardware.md](wiki/signals-and-hardware.md)
- [vm-z3-sandbox-and-game-basics.md](wiki/vm-z3-sandbox-and-game-basics.md)
- [vmp-client-server-smc-rc4-recovery.md](wiki/vmp-client-server-smc-rc4-recovery.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](wiki/windows-kernel-ioctl-hidden-feedback-maze.md)

### Forensics

- [3d-printing.md](wiki/3d-printing.md)
- [audio-frequency-and-archive-stego.md](wiki/audio-frequency-and-archive-stego.md)
- [blockchain-and-transaction-forensics.md](wiki/blockchain-and-transaction-forensics.md)
- [disk-memory-vm-and-container-forensics.md](wiki/disk-memory-vm-and-container-forensics.md)
- [file-signatures-and-flag-artifact-hunting.md](wiki/file-signatures-and-flag-artifact-hunting.md)
- [filesystem-archive-recovery-and-repair.md](wiki/filesystem-archive-recovery-and-repair.md)
- [filesystems-memory-dumps-and-raid.md](wiki/filesystems-memory-dumps-and-raid.md)
- [image-bitplane-qr-and-jpeg-stego.md](wiki/image-bitplane-qr-and-jpeg-stego.md)
- [keyboard-mouse-audio-and-physical-puzzles.md](wiki/keyboard-mouse-audio-and-physical-puzzles.md)
- [linux-git-browser-and-container-forensics.md](wiki/linux-git-browser-and-container-forensics.md)
- [network-covert-auth-and-reassembly.md](wiki/network-covert-auth-and-reassembly.md)
- [pdf-png-gif-and-text-stego.md](wiki/pdf-png-gif-and-text-stego.md)
- [peripheral-capture.md](wiki/peripheral-capture.md)
- [rf-sdr.md](wiki/rf-sdr.md)
- [video-document-and-media-stego.md](wiki/video-document-and-media-stego.md)
- [windows-registry-logs-and-credentials.md](wiki/windows-registry-logs-and-credentials.md)

### Malware

- [malware-c2-session-key-and-protocol-recovery.md](wiki/malware-c2-session-key-and-protocol-recovery.md)
- [pe-and-dotnet.md](wiki/pe-and-dotnet.md)
- [powershell-staged-payload-and-clipboard-phishing.md](wiki/powershell-staged-payload-and-clipboard-phishing.md)
- [scripts-and-obfuscation.md](wiki/scripts-and-obfuscation.md)

### Misc

- [bashjails.md](wiki/bashjails.md)
- [bgp-rpki-route-hijack.md](wiki/bgp-rpki-route-hijack.md)
- [dns.md](wiki/dns.md)
- [encodings-qr-and-esolangs.md](wiki/encodings-qr-and-esolangs.md)
- [exotic-encodings-and-file-formats.md](wiki/exotic-encodings-and-file-formats.md)
- [interactive-containers-jails-and-solvers.md](wiki/interactive-containers-jails-and-solvers.md)
- [oracles-recurrences-captcha-polyglots.md](wiki/oracles-recurrences-captcha-polyglots.md)
- [pyjails.md](wiki/pyjails.md)
- [source-backdoors-and-restricted-shell-tricks.md](wiki/source-backdoors-and-restricted-shell-tricks.md)

### OSINT

- [geolocation-and-media.md](wiki/geolocation-and-media.md)
- [osint-account-public-media-correlation.md](wiki/osint-account-public-media-correlation.md)
- [web-and-dns.md](wiki/web-and-dns.md)

### Pentest

- [linux-privesc.md](wiki/linux-privesc.md)
- [pentest-attack-chains-and-tunneling.md](wiki/pentest-attack-chains-and-tunneling.md)

## Raw 资料统计

| Direction | Markdown |
|---|---:|
| ai-ml | 9 |
| crypto | 79 |
| forensics | 19 |
| malware | 3 |
| misc | 68 |
| osint | 5 |
| pentest | 2 |
| pwn | 86 |
| reverse | 113 |
| web | 99 |

## 维护入口

- [AGENTS.md](AGENTS.md) — 维护规则、Ingest / Lint 流程和 raw/wiki 分层。
- [log.md](log.md) — 结构性维护动作、取舍和校验记录。
- [raw/](raw/) — 原始资料层，按方向归档。
- [wiki/](wiki/) — active 技巧图谱，保持扁平结构。
