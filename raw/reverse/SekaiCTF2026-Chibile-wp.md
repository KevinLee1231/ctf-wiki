# Chibile

## 题目简述

题目提供 Windows 游戏 `chibile.exe`、用户态反作弊模块 `eac_nocrt.dll`、内核驱动 `eac_shield.sys` 和启动器。客户端必须先复现游戏、DLL、驱动之间的 gate/attestation 链，才能从服务端取得两份会话密文；再分别解出用户态与内核态的 12 字符秘密，提交后获得 flag 图片。

这不是必须真实加载驱动的环境题。官方 `solve.py` 已表明：只要从二进制恢复固定常量和协议算法，就能离线构造两层证明并直接完成网络交互。`cheat/` 则提供另一条更贴近游戏流程的路径——通过自制驱动附加进程、冻结倒计时并从内存取回秘密。

## 解题过程

### 恢复 gate 链

先解析 `chibile.exe` 的 PE 节表，取第一个带 `IMAGE_SCN_CNT_CODE` 标志的节，并对其前 $\min(\text{VirtualSize},\text{SizeOfRawData})$ 字节计算 SHA-256。结合二进制中恢复出的域分隔符、CPUID 厂商值和固定密钥，证明链为：

$$
\begin{aligned}
G_{\text{game}}&=\operatorname{HMAC}_{\texttt{CHIBILE-GATE-V1}}
  (H_{\text{game}},\text{EBX},\text{ECX},\text{EDX}),\\
T_0&=\operatorname{HMAC}_{G_{\text{game}}}(\texttt{secret-room}),\\
G_{\text{dll}}&=\operatorname{HMAC}_{\texttt{CHIBILE-GATE-V1}}
  (T_0,R_1,BK_{\text{UM}},0^{32},\text{EBX},\text{ECX},\text{EDX}),\\
A_{\text{UM}}&=\operatorname{HMAC}_{G_{\text{dll}}}
  (\texttt{UM-ATTEST}\mathbin\Vert N),\\
G_{\text{drv}}&=\operatorname{HMAC}_{\texttt{CHIBILE-GATE-V1}}
  (A_{\text{UM}},BK_{\text{KM}}),\\
A_{\text{KM}}&=\operatorname{HMAC}_{G_{\text{drv}}}
  (\texttt{KM-ATTEST}\mathbin\Vert N).
\end{aligned}
$$

其中整数均按小端序编码，CPUID leaf 0 固定为 `GenuineIntel`；$R_1$ 是 DLL 运行时 `.text` 的 SHA-256，$N$ 是本次请求的 16 字节 nonce。`BK_UM`、`BK_KM` 和 $R_1$ 均可从两个反作弊二进制中恢复，官方求解脚本把它们写成已逆向得到的常量。

### 恢复两份会话秘密

服务端第一阶段验证 `client_build`、时间戳、nonce 防重放以及两份 attestation，通过后返回：

```json
{"sid":"...","c_um":"...","lh_um":"...","c_km":"...","lh_km":"..."}
```

对每一份秘密，先从二进制内置的基础半密钥 $BK$ 和服务端返回的实时半密钥 $LH$ 派生：

$$
DK=\operatorname{HMAC\!-\!SHA256}_{BK}(LH),
\qquad
KS_t(b)=\operatorname{HMAC\!-\!SHA256}_{DK}(t\mathbin\Vert\operatorname{u32le}(b)).
$$

令 $b=\lfloor i/32\rfloor$、$j=i\bmod 32$，两种解密变换分别为：

$$
\begin{aligned}
S_{\text{UM}}[i]
&=\operatorname{rotl}_8\!\left(C[i]\oplus KS_{\texttt{UM-KS}}(b)[j],3\right)
  \oplus BK[j],\\
S_{\text{KM}}[i]
&=\operatorname{rotr}_8\!\left(C[i]\oplus DK[(7i)\bmod 32],3\right)
  \oplus KS_{\texttt{KM-KS}}(b)[j].
\end{aligned}
$$

`extract_bk.py` 演示了不知道 $BK$ 时的可验证恢复方法：在 F8 壳运行后对模块做内存转储，以每个连续 32 字节窗口为候选 $BK$，结合一次捕获到的 $(C,LH)$ 解密。正确候选会得到长度为 12、字符全部属于 `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` 的明文。用户态变换直接使用 $BK$，内核态变换则通过 $DK$ 间接受它约束。

### 复现传输并提交

请求帧的结构是：

```text
u32le(length) ||
CBL2 || nonce16 || RSA-OAEP(aes_key || mac_key || xor_key || nonce)
     || iv16 || AES-CBC-PKCS7(XOR-HMAC(json)) || HMAC-SHA256
```

最内层异或流的第 $k$ 个分组为
$\operatorname{HMAC}_{xk}(N\mathbin\Vert\operatorname{u64le}(k))$。RSA 使用二进制内置的 2048 位公钥和 OAEP-SHA256；响应以 `CBR2` 开头，并继续使用本次请求的三把随机密钥验签、解密。

第一阶段响应的明文魔数为 `CBM1`。解出两个秘密后，使用新的 nonce 提交 `sid`、`typed_um`、`typed_km`、时间戳和固定版本串 `CHIBILE-R3-20260615`。成功响应以 `CBF2` 开头，后续字节是 PNG；`solve.py` 将其保存为 `flag.png`，图中内容为：

```text
SEKAI{Th4nk5_F0R_Pl4y1ng_My_Chibi_G4m3__5t4y_tun3d_f0r_my_n3xt_0n3}
```

## 方法总结

本题的决定性障碍是多模块共同实现的私有协议，而不是驱动利用。先把 gate、attestation、秘密 KDF 和外层传输拆成四层，标清每层的密钥、nonce 与字节序，便可逐层验证。遇到壳内常量时，不必只依赖静态搜索；“内存中的 32 字节窗口能否把真实会话密文解成受限字符集”本身就是很强的密钥判定 oracle。
