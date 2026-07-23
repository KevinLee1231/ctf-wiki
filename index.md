# CTF Wiki Index

这是 `ctf-*` skills 的外接技巧知识图谱。它不是默认上下文；只有当 `SKILL.md` 的直链页面不足、需要跨方向关联、详细工具说明或原始资料出处时才读取。

- Knowledge base path: `D:/文档/markdown文件/ctf-wiki`
- Structure: flat graph
- Updated: 2026-07-23

## 查询顺序

```text
已激活 ctf-* skill
  -> SKILL.md
  -> SKILL.md 中的 wiki 直链 / <direction>-tooling.md
  -> 如不足，再查本 index
  -> 对应的 wiki 页面
  -> 必要时回查 raw/<direction>/
```

## 页面类型

`wiki/` 只使用三类页面：

| Type | 作用 | raw 关系 |
|---|---|---|
| `family` | 分流页、技巧族入口、跨技巧 map。负责连接多个 technique，并说明首轮判断、变体 pivot 和失败后转向。 | 可以挂 raw，尤其是承载大量 WP case 的方向入口页。 |
| `technique` | 具体可复用技巧、攻击模式、恢复路径或判断模型。负责适用场景、识别信号、最小证据、解法骨架和常见坑。 | 可以挂 raw，作为案例和证据来源。 |
| `tooling` | 工具入口、环境限制、稳定调用方式和常见失败状态。 | 只挂工具资料或与工具调用直接相关的 raw。 |

## 方向入口速查

当题目方向已经初步明确时，优先从下表入口进入对应 family，再按页面内“路由表 / 分流流程 / 关联技巧”跳转；只有工具调用、环境路径或失败状态不清楚时再读 tooling 页。

| 方向 | 首轮入口 | 工具入口 | 适用边界 |
|---|---|---|---|
| AI / ML | [ml-model-inference-extraction-and-weight-analysis.md](wiki/ml-model-inference-extraction-and-weight-analysis.md)、[adversarial-ml.md](wiki/adversarial-ml.md)、[llm-attacks.md](wiki/llm-attacks.md) | [ai-ml-tooling.md](wiki/ai-ml-tooling.md) | 模型权重、logits、query extraction、adversarial/poisoning、prompt/tool 注入。 |
| Crypto | [encodings-qr-and-esolangs.md](wiki/encodings-qr-and-esolangs.md)、[crypto-parameter-triage-family.md](wiki/crypto-parameter-triage-family.md) | [crypto-tooling.md](wiki/crypto-tooling.md) | Base/hex/URL/ROT、字符编码、自定义码表、多层可逆编码，以及参数、密文、oracle、PRNG、签名、格/数论/对称模式。 |
| Blockchain | [blockchain-smart-contract-exploitation.md](wiki/blockchain-smart-contract-exploitation.md)、[blockchain-and-transaction-forensics.md](wiki/blockchain-and-transaction-forensics.md) | [crypto-tooling.md](wiki/crypto-tooling.md)、[web-tooling.md](wiki/web-tooling.md) | 合约状态、交易、存储布局、EVM/链运行时和链上证据；普通 dApp 漏洞仍回 Web。 |
| Web | [web-first-pass-triage-and-chain-patterns.md](wiki/web-first-pass-triage-and-chain-patterns.md) | [web-tooling.md](wiki/web-tooling.md) | HTTP 应用、认证、浏览器 bot、解析器差异、内部服务或漏洞链。 |
| Cloud / Infra | [oauth-saml-cors-and-cicd.md](wiki/oauth-saml-cors-and-cicd.md)、[pentest-attack-chains-and-tunneling.md](wiki/pentest-attack-chains-and-tunneling.md) | [web-tooling.md](wiki/web-tooling.md)、[pentest-tooling.md](wiki/pentest-tooling.md) | IAM、云控制面、资源策略、Serverless、编排、IaC、CI/CD 和供应链。 |
| Pwn | [pwn-first-pass-red-flags-and-protections.md](wiki/pwn-first-pass-red-flags-and-protections.md) | [pwn-tooling.md](wiki/pwn-tooling.md) | 二进制、libc/heap/ROP、kernel、sandbox、JIT/VM primitive 和保护组合。 |
| Reverse | [reverse-first-pass-workflow-and-debugging.md](wiki/reverse-first-pass-workflow-and-debugging.md) | [reverse-tooling.md](wiki/reverse-tooling.md) | 载体识别、语言运行时、VM/obfuscation、平台/固件/硬件和动态调试。 |
| Mobile | [mobile-firmware-kernel-and-game-re.md](wiki/mobile-firmware-kernel-and-game-re.md)、[android-games-hardware-and-runtime-platforms.md](wiki/android-games-hardware-and-runtime-platforms.md) | [reverse-tooling.md](wiki/reverse-tooling.md) | Android/iOS 组件、IPC、权限、签名、Keystore/Keychain 与平台运行时；普通 APK 算法还原仍回 Reverse。 |
| Hardware / Embedded | [hardware-isa-bootloader-and-kvm.md](wiki/hardware-isa-bootloader-and-kvm.md)、[signals-and-hardware.md](wiki/signals-and-hardware.md) | [reverse-tooling.md](wiki/reverse-tooling.md)、[forensics-tooling.md](wiki/forensics-tooling.md) | JTAG/UART、总线、MCU、RF、侧信道、Secure Boot 与硬件信号。 |
| Forensics | [cross-domain-forensics-technique-map.md](wiki/cross-domain-forensics-technique-map.md) | [forensics-tooling.md](wiki/forensics-tooling.md) | 文件、PCAP、磁盘/内存、日志、外设采集和容器证据恢复。 |
| Stego | [image-bitplane-qr-and-jpeg-stego.md](wiki/image-bitplane-qr-and-jpeg-stego.md)、[audio-frequency-and-archive-stego.md](wiki/audio-frequency-and-archive-stego.md)、[video-document-and-media-stego.md](wiki/video-document-and-media-stego.md) | [forensics-tooling.md](wiki/forensics-tooling.md) | 图像、音频、视频、文档、QR 碎片、视觉/空间线索、游戏场景或隐蔽信道中隐藏信息的检测、重组与提取。 |
| Cross-category（调度状态） | [cross-category-triage-family.md](wiki/cross-category-triage-family.md) | [cross-category-tooling.md](wiki/cross-category-tooling.md) | 证据不足时由 `ctf-solve-challenge` 继续首轮分流；不是 raw 一级方向或执行型专项。 |
| OSINT | [osint-account-public-media-correlation.md](wiki/osint-account-public-media-correlation.md)、[geolocation-and-media.md](wiki/geolocation-and-media.md)、[web-and-dns.md](wiki/web-and-dns.md) | [osint-tooling.md](wiki/osint-tooling.md) | 公开账号、地理/媒体、DNS/历史网页和可复查公开证据链。 |
| Pentest | [pentest-attack-chains-and-tunneling.md](wiki/pentest-attack-chains-and-tunneling.md)、[linux-privesc.md](wiki/linux-privesc.md) | [pentest-tooling.md](wiki/pentest-tooling.md) | 凭据、服务枚举、隧道、横向移动和 Linux 本机提权。 |
| Malware | [scripts-and-obfuscation.md](wiki/scripts-and-obfuscation.md)、[pe-and-dotnet.md](wiki/pe-and-dotnet.md)、[malware-c2-session-key-and-protocol-recovery.md](wiki/malware-c2-session-key-and-protocol-recovery.md) | [malware-tooling.md](wiki/malware-tooling.md) | 脚本/包载荷、PE/.NET、shellcode、配置提取、C2 协议和会话恢复。 |

## Family Pages

| Family | 方向 | 用途 |
|---|---|---|
| [adversarial-ml.md](wiki/adversarial-ml.md) | ai-ml | 对抗样本、evasion、patch、poisoning、backdoor 和梯度/查询反馈分流。 |
| [llm-attacks.md](wiki/llm-attacks.md) | ai-ml | Prompt injection、jailbreak、token smuggling、LLM 输出派生 key/seed/password 和工具调用分流。 |
| [ml-model-inference-extraction-and-weight-analysis.md](wiki/ml-model-inference-extraction-and-weight-analysis.md) | ai-ml | 模型推理、query extraction、model inversion、weight diff、encoder collision 和 membership inference 分流。 |
| [block-mode-misuse-family.md](wiki/block-mode-misuse-family.md) | crypto | 分组模式、MAC、oracle 和对称密码误用分流。 |
| [crypto-parameter-triage-family.md](wiki/crypto-parameter-triage-family.md) | crypto | RSA/ECC/格/PRNG/哈希/代数等 crypto 参数首轮分流。 |
| [rsa-attacks.md](wiki/rsa-attacks.md) | crypto | RSA 小指数、共模、广播、Wiener、Fermat、多素数和基础 oracle 分流。 |
| [rsa-specialized-structures-and-oracles.md](wiki/rsa-specialized-structures-and-oracles.md) | crypto | RSA 特殊素数结构、CRT 泄露、同态签名、fault、ROCA 和复杂 oracle 分流。 |
| [hash-protocol-and-oracle-attacks.md](wiki/hash-protocol-and-oracle-attacks.md) | crypto | Hash/MAC、CRC、compression、padding、协议 proof 和 oracle 反馈分流。 |
| [classical-xor-and-substitution-ciphers.md](wiki/classical-xor-and-substitution-ciphers.md) | crypto | 古典替换、Vigenere、XOR、many-time pad、文件头推 key 和视觉轻量 cipher 分流。 |
| [lattice-and-lwe.md](wiki/lattice-and-lwe.md) | crypto | HNP、截断 LCG、LWE/Ring-LWE、CVP/SVP 和 subset sum 分流。 |
| [number-theory-and-algebra-attacks.md](wiki/number-theory-and-algebra-attacks.md) | crypto | DLP、Coppersmith、p-adic、GF(2) 和特殊代数结构分流。 |
| [homomorphic-and-exotic-algebra.md](wiki/homomorphic-and-exotic-algebra.md) | crypto | 同态 oracle、Paillier/ElGamal、低频代数和协议可塑性分流。 |
| [zkp-secret-sharing-and-proof-systems.md](wiki/zkp-secret-sharing-and-proof-systems.md) | crypto | ZKP、garbled circuit、Shamir、KZG/pairing 和证明系统分流。 |
| [exotic-secret-sharing-rabin-and-polynomials.md](wiki/exotic-secret-sharing-rabin-and-polynomials.md) | crypto | BIP39、Asmuth-Bloom、Rabin、多项式、Vandermonde 和长尾原语分流。 |
| [blockchain-smart-contract-exploitation.md](wiki/blockchain-smart-contract-exploitation.md) | blockchain | EVM、Solana/Anchor、Sui Move、链上随机、proxy/delegatecall、ABI 和 capability/resource 分流。 |
| [mt-lcg-and-seed-recovery.md](wiki/mt-lcg-and-seed-recovery.md) | crypto/web | MT19937、LCG、C rand、V8 Math.random、时间种子、token/key 和 PRNG 状态恢复分流。 |
| [prng-z3-lcg-and-timing-attacks.md](wiki/prng-z3-lcg-and-timing-attacks.md) | crypto | 约束、Z3、partial output、timing oracle 和跨域 PRNG 恢复分流。 |
| [rc4-lfsr-and-keystream-reuse.md](wiki/rc4-lfsr-and-keystream-reuse.md) | crypto | LFSR、RC4、keystream reuse、known plaintext 和流密码恢复分流。 |
| [ecc-dlp-and-signature-attacks.md](wiki/ecc-dlp-and-signature-attacks.md) | crypto | ECC/DLP、small subgroup、invalid curve、nonce reuse 和签名恢复分流。 |
| [sqli-filter-and-oracle-family.md](wiki/sqli-filter-and-oracle-family.md) | web | SQLi 过滤、盲注、二阶注入和 oracle 分流。 |
| [web-first-pass-triage-and-chain-patterns.md](wiki/web-first-pass-triage-and-chain-patterns.md) | web | Web 首轮业务流、解析差异和漏洞链分流。 |
| [auth-bypass-cookies-and-hidden-routes.md](wiki/auth-bypass-cookies-and-hidden-routes.md) | web | Cookie/session、隐藏路由、代理 ACL、OAuth 和访问控制分流。 |
| [auth-jwt.md](wiki/auth-jwt.md) | web | JWT/JWE、签名 cookie、算法混淆、key lookup 和 token 伪造分流。 |
| [auth-edge-cases-and-protocol-bypasses.md](wiki/auth-edge-cases-and-protocol-bypasses.md) | web | Hash bucket、Unicode、SRP/DH、AQL/NoSQL 等长尾认证边界分流。 |
| [oauth-saml-cors-and-cicd.md](wiki/oauth-saml-cors-and-cicd.md) | web/cloud-infra | OAuth/OIDC、SAML、CORS、Git/CI 凭据和身份平台链分流。 |
| [php-lfi-ssti-ssrf-and-type-juggling.md](wiki/php-lfi-ssti-ssrf-and-type-juggling.md) | web | PHP/LFI/SSTI/SSRF/XXE/type juggling 等服务端解释层差异分流。 |
| [path-traversal-ssrf-upload-and-rsc.md](wiki/path-traversal-ssrf-upload-and-rsc.md) | web | 路径穿越、上传、SSRF、渲染器、RSC 和内部服务链分流。 |
| [polyglot-url-tricks-and-ssrf-leaks.md](wiki/polyglot-url-tricks-and-ssrf-leaks.md) | web | URL parser 差异、polyglot 上传、CRLF/gopher 和 SSRF 凭据泄露分流。 |
| [ruby-php-upload-and-ssti-rce.md](wiki/ruby-php-upload-and-ssti-rce.md) | web | 语言 eval、模板、上传、反序列化和命令执行链分流。 |
| [sqli-upload-deser-and-command-rce.md](wiki/sqli-upload-deser-and-command-rce.md) | web | SQLi、上传、反序列化、命令包装器和文件读到 RCE 的链路分流。 |
| [known-cves-and-n-day-exploits.md](wiki/known-cves-and-n-day-exploits.md) | web | CVE/N-day 版本边界、最小 PoC 和后续漏洞链分流。 |
| [xss-dom-and-browser-tricks.md](wiki/xss-dom-and-browser-tricks.md) | web | XSS、DOM、admin bot、缓存/MIME、CSP 和浏览器外带分流。 |
| [xml-command-and-graphql-injection.md](wiki/xml-command-and-graphql-injection.md) | web | XXE/XML、命令注入、GraphQL 和二次解释型注入分流。 |
| [parser-wrapper-and-legacy-ssrf-tricks.md](wiki/parser-wrapper-and-legacy-ssrf-tricks.md) | web | 解析器差异、wrapper、legacy SSRF 和内部资源二段加载分流。 |
| [php-java-python-deserialization.md](wiki/php-java-python-deserialization.md) | web | Java/Python/PHP/.NET 反序列化、gadget 和包装层分流。 |
| [node-and-prototype.md](wiki/node-and-prototype.md) | web | Node 原型污染、gadget、VM/sandbox escape、Happy-DOM 和 RSC 转向分流。 |
| [pwn-first-pass-red-flags-and-protections.md](wiki/pwn-first-pass-red-flags-and-protections.md) | pwn | Pwn 首轮保护、漏洞族和最短利用路线分流。 |
| [oob-jit-parser-primitives-family.md](wiki/oob-jit-parser-primitives-family.md) | pwn | OOB/JIT/parser primitive 分流。 |
| [overflow-basics.md](wiki/overflow-basics.md) | pwn | 栈溢出、基础 OOB、canary、signed index 和低熵覆盖分流。 |
| [linux-kernel-exploit-basics.md](wiki/linux-kernel-exploit-basics.md) | pwn | Linux kernel pwn 环境、primitive、保护和提权目标分流。 |
| [cross-primitive-escape-and-hybrid-exploit-map.md](wiki/cross-primitive-escape-and-hybrid-exploit-map.md) | pwn | 多 primitive、沙箱和混合利用路线 map。 |
| [seccomp-ret2dlresolve-and-runtime-primitives.md](wiki/seccomp-ret2dlresolve-and-runtime-primitives.md) | pwn | Seccomp、ORW、ret2dlresolve、runtime syscall 和动态解析分流。 |
| [interpreter-jit-canary-and-integer-exploits.md](wiki/interpreter-jit-canary-and-integer-exploits.md) | pwn | 解释器/JIT/整数/GC/canary/运行时 primitive 分流。 |
| [runtime-protection-and-tls-exploits.md](wiki/runtime-protection-and-tls-exploits.md) | pwn | FILE/TLS/atexit/shadow stack/heap leak 和保护绕过分流。 |
| [stack-pivots-srop-and-seccomp-rop.md](wiki/stack-pivots-srop-and-seccomp-rop.md) | pwn | 栈迁移、SROP、vDSO/vsyscall、JIT-ROP 和 constrained ROP 分流。 |
| [windows-arm-and-cross-platform-exploits.md](wiki/windows-arm-and-cross-platform-exploits.md) | pwn | Windows、ARM、SEH/CFG、跨架构 shellcode 和平台约束分流。 |
| [ret2csu-dynelf-and-shellcode.md](wiki/ret2csu-dynelf-and-shellcode.md) | pwn | ret2libc、ret2csu、DynELF、badchar ROP 和 shellcode 落地分流。 |
| [heap-houses-unlink-and-tcache.md](wiki/heap-houses-unlink-and-tcache.md) | pwn | Heap house、unlink、tcache、largebin 和 allocator metadata 分流。 |
| [heap-uaf-tcache-and-custom-allocator.md](wiki/heap-uaf-tcache-and-custom-allocator.md) | pwn | UAF、double free、tcache poisoning、custom allocator 和对象生命周期 primitive 分流。 |
| [heap-fsop-file-structure-attacks.md](wiki/heap-fsop-file-structure-attacks.md) | pwn | glibc FILE、stdout/stdin、FSOP、IO buffer 和 FILE 结构落点分流。 |
| [kernel-uaf-race-and-slab-techniques.md](wiki/kernel-uaf-race-and-slab-techniques.md) | pwn | Kernel UAF、race、SLUB/slab、BPF 和 PTE/page overlap 分流。 |
| [python-vm-and-proc-sandbox-escape.md](wiki/python-vm-and-proc-sandbox-escape.md) | pwn | Python/VM、/proc、FUSE/CUSE、fifo、emulator 和 sandbox primitive 分流。 |
| [pentest-attack-chains-and-tunneling.md](wiki/pentest-attack-chains-and-tunneling.md) | pentest/cloud-infra | Foothold、凭据、隧道、横向、云身份和提权 attack graph 分流。 |
| [linux-privesc.md](wiki/linux-privesc.md) | pentest | Linux 单机提权、服务滥用、备份凭据和本机 pivot 分流。 |
| [reverse-first-pass-workflow-and-debugging.md](wiki/reverse-first-pass-workflow-and-debugging.md) | reverse | 逆向首轮载体、调试、dump 和比较点分流。 |
| [vm-obfuscation-transform-family.md](wiki/vm-obfuscation-transform-family.md) | reverse | VM、字节码、变换链和解释器类题分流。 |
| [android-games-hardware-and-runtime-platforms.md](wiki/android-games-hardware-and-runtime-platforms.md) | mobile/reverse/hardware-embedded | Android、游戏资源、Electron/Node、硬件抽象和运行时平台分流。 |
| [mobile-firmware-kernel-and-game-re.md](wiki/mobile-firmware-kernel-and-game-re.md) | mobile/reverse/hardware-embedded | Mach-O/iOS、固件、驱动、eBPF、游戏引擎和 CAN 环境分流。 |
| [go-rust-jvm-and-cpp-reversing.md](wiki/go-rust-jvm-and-cpp-reversing.md) | reverse | Go、Rust、JVM、C++、Swift/Kotlin 等语言运行时分流。 |
| [python-bytecode-esolangs-and-uefi.md](wiki/python-bytecode-esolangs-and-uefi.md) | reverse | Python 字节码、Pyarmor/Nuitka、esolang、UEFI 和低频字节码分流。 |
| [loader-vm-image-and-kernel-patterns.md](wiki/loader-vm-image-and-kernel-patterns.md) | reverse | Loader、镜像/bitmap、kernel module、binfmt 和二阶段执行链分流。 |
| [font-shader-firmware-and-legacy-patterns.md](wiki/font-shader-firmware-and-legacy-patterns.md) | reverse | 字体、shader、BPF、MBR、legacy 和 side-channel 载体分流。 |
| [anti-analysis.md](wiki/anti-analysis.md) | reverse | 反调试、anti-VM、anti-DBI、自校验和反反汇编分流。 |
| [packers-deobfuscation-and-debug-automation.md](wiki/packers-deobfuscation-and-debug-automation.md) | reverse | 壳、商业虚拟化、动态解密、patch、trace 和降维路线分流。 |
| [self-decrypting-strings-and-lattice-patterns.md](wiki/self-decrypting-strings-and-lattice-patterns.md) | reverse | 自解密、字符串恢复、格/线性约束和魔改 cipher 恢复分流。 |
| [hardware-isa-bootloader-and-kvm.md](wiki/hardware-isa-bootloader-and-kvm.md) | hardware-embedded/reverse/pwn | 低频 ISA、固件、bootloader、KVM、MCU、TrustZone 和协处理器分流。 |
| [runtime-patching-oracles-and-tracing.md](wiki/runtime-patching-oracles-and-tracing.md) | reverse | 运行时 patch、oracle、hook、trace、coredump 和上下文替换分流。 |
| [signal-trace-and-packed-anti-analysis.md](wiki/signal-trace-and-packed-anti-analysis.md) | reverse | 信号处理器、trace 反演、父子进程 dump 和 packed module 分流。 |
| [scripts-and-obfuscation.md](wiki/scripts-and-obfuscation.md) | malware | JS/PowerShell/SVG/包载荷、脚本混淆和恶意载荷链分流。 |
| [pe-and-dotnet.md](wiki/pe-and-dotnet.md) | malware | PE/.NET、配置提取、DNS C2、PyInstaller/PyArmor、shellcode 和内存证据分流。 |
| [cross-domain-forensics-technique-map.md](wiki/cross-domain-forensics-technique-map.md) | forensics | 跨 PCAP/磁盘/内存/媒体/容器取证的下一跳 map。 |
| [file-signatures-and-flag-artifact-hunting.md](wiki/file-signatures-and-flag-artifact-hunting.md) | forensics | 未知文件、magic/metadata、尾随数据、嵌入对象和常见编码层首检分流。 |
| [filesystem-archive-recovery-and-repair.md](wiki/filesystem-archive-recovery-and-repair.md) | forensics | 文件系统删除恢复、损坏归档、ZipCrypto 已知明文、加密容器和数据库页恢复分流。 |
| [pcap-protocol-credential-recovery-family.md](wiki/pcap-protocol-credential-recovery-family.md) | forensics | PCAP、协议重组、凭据和 key 恢复分流。 |
| [windows-registry-logs-and-credentials.md](wiki/windows-registry-logs-and-credentials.md) | forensics | Windows 注册表、事件日志、NTFS、浏览器凭据和反取证时间线分流。 |
| [pdf-png-gif-and-text-stego.md](wiki/pdf-png-gif-and-text-stego.md) | stego | PDF、PNG、GIF、SVG、文本和容器媒体隐写分流。 |
| [linux-git-browser-and-container-forensics.md](wiki/linux-git-browser-and-container-forensics.md) | forensics | Linux 日志、Git 对象库、浏览器 profile、Docker layer 和凭据恢复分流。 |
| [image-bitplane-qr-and-jpeg-stego.md](wiki/image-bitplane-qr-and-jpeg-stego.md) | stego | JPEG/PNG/BMP/GIF、bitplane、QR 重组、视觉/空间线索和图像隐写分流。 |
| [disk-memory-vm-and-container-forensics.md](wiki/disk-memory-vm-and-container-forensics.md) | forensics | 磁盘、内存、VM、容器、Android 和云存储载体分流。 |
| [filesystems-memory-dumps-and-raid.md](wiki/filesystems-memory-dumps-and-raid.md) | forensics | 分区/文件系统、minidump、VMDK sparse、RAID 和卷恢复分流。 |
| [network-covert-auth-and-reassembly.md](wiki/network-covert-auth-and-reassembly.md) | forensics | 网络 covert channel、凭据恢复、协议重组和 RTP 音频分流。 |
| [signals-and-hardware.md](wiki/signals-and-hardware.md) | hardware-embedded/forensics | 显示链路、总线、RF、功耗、键盘声学和硬件信号恢复分流。 |
| [audio-frequency-and-archive-stego.md](wiki/audio-frequency-and-archive-stego.md) | stego | 音频频域、SSTV、DTMF、声道 LSB、DeepSound、语音认证和音频 archive 分流。 |
| [video-document-and-media-stego.md](wiki/video-document-and-media-stego.md) | stego | 视频帧、文档对象、媒体容器、JXL/PDF/EXIF 和运行时媒体 dump 分流。 |
| [peripheral-capture.md](wiki/peripheral-capture.md) | forensics | USB HID、鼠标/键盘、LED Morse、Bluetooth RFCOMM 和外设 report 重组分流。 |
| [keyboard-mouse-audio-and-physical-puzzles.md](wiki/keyboard-mouse-audio-and-physical-puzzles.md) | hardware-embedded/stego/forensics | USB HID、鼠标/键盘、音频频率、视频轨迹和物理动作记录恢复分流。 |
| [encodings-qr-and-esolangs.md](wiki/encodings-qr-and-esolangs.md) | crypto | Base/hex/URL/ROT、字符编码、自定义码表、二维码载荷、esolang 和多层可逆编码链分流。 |
| [pyjails.md](wiki/pyjails.md) | pwn/web/reverse | Python 受限执行、Web 表达式沙箱、对象链、字符集、oracle 和 agent sandbox 分流。 |
| [vm-z3-sandbox-and-game-basics.md](wiki/vm-z3-sandbox-and-game-basics.md) | cross-category/reverse/pwn/cloud-infra | VM/解释器、WASM/游戏、Z3/约束、K8s、浮点、sandbox 和 VM primitive 分流。 |
| [game-state-websocket-and-wasm.md](wiki/game-state-websocket-and-wasm.md) | cross-category/web/reverse/stego | 游戏状态、WebSocket、session、WASM linear memory、资源包和场景隐藏线索分流。 |
| [cross-category-triage-family.md](wiki/cross-category-triage-family.md) | cross-category | 游戏、shell、oracle、pyjail、轻量附件及多方向混合题分流。 |
| [dns.md](wiki/dns.md) | web/forensics/osint/pentest/malware | DNSSEC、zone transfer、rebinding、DNS tunnel、SPF/TXT 和解析链分流。 |
| [file-triage-archives-and-one-liners.md](wiki/file-triage-archives-and-one-liners.md) | forensics/cross-category | 文件首检、压缩包、一行式和轻量附件 triage。 |
| [oracles-recurrences-captcha-polyglots.md](wiki/oracles-recurrences-captcha-polyglots.md) | cross-category | 比较/timeout oracle、递推、CAPTCHA、QR 结构约束和 polyglot 分流。 |
| [interactive-containers-jails-and-solvers.md](wiki/interactive-containers-jails-and-solvers.md) | cross-category/pwn/pentest/cloud-infra/reverse | 交互服务、容器边界、jail 变体和小型求解器分流。 |
| [source-backdoors-and-restricted-shell-tricks.md](wiki/source-backdoors-and-restricted-shell-tricks.md) | pwn/pentest/web | 源码后门、受限 shell、HISTFILE/`bash -v`、rvim 插件和工具副作用分流。 |
| [bashjails.md](wiki/bashjails.md) | pwn/pentest | Bash/rbash 受限 shell、字符集、输出通道和 post-shell 分流。 |
| [exotic-encodings-and-file-formats.md](wiki/exotic-encodings-and-file-formats.md) | cross-category | 低频编码、结构化格式、条码/音频映射和解析器差异的跨方向分流。 |
| [geolocation-and-media.md](wiki/geolocation-and-media.md) | osint | 图片、视频、坐标、街景、路牌、地标、IP 和媒体地理定位分流。 |
| [web-and-dns.md](wiki/web-and-dns.md) | osint | 搜索引擎、公开文档、DNS、WHOIS、Wayback、Shodan 和公开仓库分流。 |
| [osint-account-public-media-correlation.md](wiki/osint-account-public-media-correlation.md) | osint | 账号、公开媒体、游戏平台、社交平台、archive 和身份链分流。 |

## Tooling Pages

| Direction | Tooling |
|---|---|
| ai-ml | [ai-ml-tooling.md](wiki/ai-ml-tooling.md) |
| crypto | [crypto-tooling.md](wiki/crypto-tooling.md) |
| forensics | [forensics-tooling.md](wiki/forensics-tooling.md) |
| malware | [malware-tooling.md](wiki/malware-tooling.md) |
| cross-category | [cross-category-tooling.md](wiki/cross-category-tooling.md) |
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

- [linear-model-input-lattice-recovery.md](wiki/linear-model-input-lattice-recovery.md)
- [linear-model-parameter-recovery.md](wiki/linear-model-parameter-recovery.md)
- [transformer-logit-inversion.md](wiki/transformer-logit-inversion.md)

### Crypto / Blockchain

- [lorenz-and-book-cipher-attacks.md](wiki/lorenz-and-book-cipher-attacks.md)

### Web

- [artifact-trust-ssrf-to-node-require-rce.md](wiki/artifact-trust-ssrf-to-node-require-rce.md)
- [csp-xsleak-and-browser-exfiltration.md](wiki/csp-xsleak-and-browser-exfiltration.md)
- [json-duplicate-key-hmac-parser-differential.md](wiki/json-duplicate-key-hmac-parser-differential.md)
- [path-confusion-to-signed-internal-request-chain.md](wiki/path-confusion-to-signed-internal-request-chain.md)
- [protocol-relay-and-internal-service-injection.md](wiki/protocol-relay-and-internal-service-injection.md)
- [workflow-runner-internal-api-chain.md](wiki/workflow-runner-internal-api-chain.md)

### Cross-Direction

- [bgp-rpki-route-hijack.md](wiki/bgp-rpki-route-hijack.md)
- [race-condition-and-concurrency-exploits.md](wiki/race-condition-and-concurrency-exploits.md)

### Pwn

- [data-interpretation-memory-primitives.md](wiki/data-interpretation-memory-primitives.md)
- [format-string.md](wiki/format-string.md)
- [kaslr-kpti-smep-and-kernel-debugging.md](wiki/kaslr-kpti-smep-and-kernel-debugging.md)

### Reverse

- [compare-breakpoint-plaintext-recovery.md](wiki/compare-breakpoint-plaintext-recovery.md)
- [embedded-python-pyd-custom-aes.md](wiki/embedded-python-pyd-custom-aes.md)
- [vmp-client-server-smc-rc4-recovery.md](wiki/vmp-client-server-smc-rc4-recovery.md)
- [windows-kernel-ioctl-hidden-feedback-maze.md](wiki/windows-kernel-ioctl-hidden-feedback-maze.md)

### Forensics

- [3d-printing.md](wiki/3d-printing.md)
- [blockchain-and-transaction-forensics.md](wiki/blockchain-and-transaction-forensics.md)
- [rf-sdr.md](wiki/rf-sdr.md)

### Malware

- [malware-c2-session-key-and-protocol-recovery.md](wiki/malware-c2-session-key-and-protocol-recovery.md)
- [powershell-staged-payload-and-clipboard-phishing.md](wiki/powershell-staged-payload-and-clipboard-phishing.md)

## Raw 资料统计

| Direction | Markdown |
|---|---:|
| _unclassified（暂存） | 6 |
| ai-ml | 29 |
| blockchain | 25 |
| cloud-infra | 5 |
| crypto | 275 |
| forensics | 68 |
| hardware-embedded | 5 |
| malware | 13 |
| mobile | 4 |
| osint | 28 |
| pentest | 15 |
| pwn | 256 |
| reverse | 258 |
| stego | 55 |
| web | 263 |
| **Total** | **1305** |

## 维护入口

- [AGENTS.md](AGENTS.md) — 维护规则、Ingest / Lint 流程和 raw/wiki 分层。
- [log.md](log.md) — 结构性维护动作、取舍和校验记录。
- [raw/](raw/) — 原始资料层，按方向归档。
- [wiki/](wiki/) — active 技巧图谱，保持扁平结构。
