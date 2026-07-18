# SUCTF2026-LightNovel

## 题目简述

附件 `suctf-ad-final-fix.pcapng` 是一段 Active Directory 网络流量。攻击者先后以 `kanna.seto` 和 `Administrator` 的身份，通过远程任务计划服务（TSCH）注册、执行并读取隐藏任务，把文件内容塞进任务 XML 的 `Description` 字段。解题需要依次处理 NTLMv2、Kerberos/GSSAPI、PKINIT、U2U PAC 凭据和 Microsoft NTP 扩展，最终恢复隐写图片并解开 flag。

这不是单纯“给 Wireshark 配一个密码”即可完成的题：三条 TSCH 会话采用不同认证方式，DCE/RPC 请求还会跨 TCP 段和 RPC fragment 分片。关键会话为：

| TCP 流 | 客户端端口 | 身份与认证 | 目标材料 |
|---|---:|---|---|
| `tcp.stream == 1` | 47354 | `kanna.seto`，NTLM | `hint.zip` |
| `tcp.stream == 5` | 34338 | `kanna.seto`，Kerberos | `cert.zip` |
| `tcp.stream == 10` | 33980 | `Administrator`，Kerberos | `flag.jpg` |

## 解题过程

### 1. NTLM 会话：恢复 `hint.zip`

先查看 TCP 会话和 TSCH 绑定流量：

```bash
tshark -r suctf-ad-final-fix.pcapng -q -z conv,tcp
tshark -r suctf-ad-final-fix.pcapng \
  -Y 'frame.number==20 || frame.number==21 || frame.number==23 || frame.number==44 || frame.number==45' -V
```

`frame 20/21/23` 是 `TaskSchedulerService` 的 NTLM negotiate、challenge 和 authenticate，认证用户为 `wire.com\kanna.seto`；后面紧接 `SchRpcRegisterTask` 与 `SchRpcRun`。从 Type 2 和 Type 3 消息中可取出：

```text
Server challenge : e9b597a6e03a5122
User / domain    : kanna.seto / wire.com
NT proof         : c4ec074163bee82d9f829d1aa22de185
```

按 NetNTLMv2 的 Hashcat 格式拼接用户名、域、challenge、NT proof 和响应 blob，以 rockyou 字典运行模式 5600：

```bash
hashcat -m 5600 netntlmv2.txt /usr/share/wordlists/rockyou.txt
```

得到口令：

```text
taylorswift<3
```

将口令交给 Wireshark/TShark 后即可直接解开这一会话的 NTLM 密封 DCE/RPC stub：

```bash
tshark -o 'ntlmssp.nt_password:taylorswift<3' \
  -r suctf-ad-final-fix.pcapng -Y 'frame.number==42 || frame.number==759' \
  -T fields -e frame.number -e dcerpc.decrypted_stub_data
```

注册请求中的 UTF-16LE 任务 XML 对应任务 `\gsmIqwfB`，其 PowerShell 动作读取 `C:\hint.zip`，再把文件字节做 Base64 后写入任务描述。脚本虽然定义了 AES-CBC 辅助函数和密钥 `7mLnyC9VW9IZ8opOl7ouNQ==`，但这条上传路径没有调用加密函数；恢复 `frame 759` 中的 `Description` 后只需 Base64 解码。ZIP 使用了口令复用，解压密码仍是 `taylorswift<3`。

`hint.zip` 给出一张提示图和文本 `404 Flag Not Found, but kanno.seto is literally peak cuteness!!!!`。图片本身没有有效隐写载荷，作用是提示继续围绕 `kanna.seto` 分析后续 Kerberos 流量。

### 2. Kerberos 会话：恢复 `cert.zip`

与 `kanna.seto` 相关的 Kerberos 关键帧为：

- `859/860`：AS-REQ / AS-REP；
- `868/869`：TGS-REQ / TGS-REP；
- `874/876/878`：Kerberos 认证的 RPC bind；
- 随后的 `tcp.stream == 5`：注册并读取第二个计划任务。

AS-REP 的 `ETYPE-INFO2` 给出 AES256 类型和盐 `WIRE.COMKanna.Seto`。用已知口令派生长期密钥：

```python
from impacket.krb5 import crypto

key = crypto.string_to_key(18, 'taylorswift<3', 'WIRE.COMKanna.Seto', None)
print(key.contents.hex())
```

结果为：

```text
1ebf62851842b93e4b095f8474a905a4fc4d315796202540019d86e6570b8ca8
```

将它写成 etype 18 的 keytab 后，TShark 能逐层解开 AS-REP、TGS-REP 和 AP-REP。`frame 876` 的 AP-REP 携带本次 GSSAPI 安全上下文使用的 acceptor subkey：

```text
6c729591c51fd38f4c462d74566eeb4a40a4511a9c85bc81232e737a98d8d1f2
```

`tcp.stream == 5` 的 RPC stub 使用 Kerberos GSSAPI AES256 密封。稳定的恢复步骤是：

1. 用 `tshark` 导出同一方向的 `tcp.seq_raw` 与 `tcp.payload`，按序号重组并去除重传重叠；
2. 依 DCE/RPC 头部的 `frag_len`、`auth_len`、`call_id` 和 FIRST/LAST 标志拆分 PDU；不能假设一帧只有一个 fragment，`frame 1025` 实际含两个；
3. 从安全尾部取 GSS Wrap token，将 `token[16:] + encrypted_stub` 按 `RRC + EC` 反向旋转；
4. 用上面的 subkey 和 AES256-CTS 解密。客户端请求使用 key usage 24，服务端响应使用 usage 22；去掉 token、EC 与 RPC padding 后再拼接明文 stub；
5. 从明文中定位 UTF-16LE 的 `<Task>...</Task>`。

注册请求跨越 `frame 881` 到 `2572`，恢复出的任务名为 `\JlWveTli`。PowerShell 中的真实逻辑是读取任务描述，以 Base64 解出 `IV || ciphertext`，再用下列密钥做 AES-CBC/PKCS#7 解密；所得明文仍是 Base64，二次解码后才是 `cert.zip`：

```text
AES key (Base64): PYake61OOYCKw0zg+oT/Qg==
target           : C:\cert.zip
```

`cert.zip` 中未加密的 `hint.txt` 写着：

```text
潮声只听开口处
The sea listens where the lines begin
```

未加密的 `cert.jpg` 用 `steghide` 可取出四句诗：

```text
濑水晚霞映海天，户外潮声入远烟。
环佩清姿临碧浪，奈何人间少此颜。
倾心落日添柔影，城畔微风动鬓边。
绝代芳华如画里，色映云霞胜月妍。
```

提示要求读取各分句开头，得到 ZIP 密码 `濑户环奈倾城绝色`。解压后获得：

- `administrator.pfx`：导入密码为空；证书 Subject 是 `CN=Kanna.seto`，但 SAN UPN 是 `administrator@wire.com`，并带有 RID 500 的 SID，因此 AD 映射目标是管理员账户；
- `wiredc.ccache`：保存 `Administrator@WIRE.COM` 的 TGT；
- `key`：管理员 PKINIT 的 32 字节 AS-REP key：

```text
01ea8c39173e5e4afbb5a6580b118e4cc21b16d399b8e2322b9090e68acd080a
```

### 3. PKINIT/U2U：恢复管理员口令

管理员认证链位于更早的 `frame 779/783/793/795`：

- `779/783` 是用上述证书进行的 PKINIT AS-REQ / AS-REP；
- `793` 是设置了 `enc-tkt-in-skey` 且携带 additional ticket 的 U2U TGS-REQ；
- `795` 是返回给 `Administrator` 的 U2U TGS-REP。

用 `key` 中的 AS-REP key、etype 18、key usage 3 解开 `frame 783` 的 AS-REP `enc-part`，得到的 TGT session key 与 `wiredc.ccache` 完全一致：

```text
e7d900a23fd982ccf1f4142a360291735e4af423e0e7255a53e6102afd27f352
```

U2U 票据由这把 TGT session key 加密。对 `frame 795` 的 ticket `enc-part` 使用 key usage 2，随后沿 `authorization-data -> AD-IF-RELEVANT -> AD-WIN2K-PAC` 找到 `PAC_CREDENTIAL_INFO`。其 `SerializedData` 还需用 AS-REP key 和 key usage 16 解密。解出的 NDR 数据中包含 `NTLM_SUPPLEMENTAL_CREDENTIAL`，其中 `NtPassword` 即管理员 NT Hash：

```text
bedcf78571904538b1919672e4521c4e
```

字典反查得到管理员口令：

```text
Taylor@1989
```

这里两把 32 字节密钥不可混用：TGT session key 用来解 U2U ticket；`key` 文件中的 PKINIT AS-REP key 用来解 `PAC_CREDENTIAL_INFO`。U2U TGS-REP 外层 `enc-part` 的 reply session key只是校验链路，并不是解 PAC 凭据所需的最终密钥。

### 4. 管理员 TSCH 与 TimeRoast：得到 flag

用 `Taylor@1989` 和盐 `WIRE.COMAdministrator` 派生管理员 AES256 长期密钥并生成 keytab，可解开最后一组认证：`2671/2681` 为 AS 流程，`2689/2691` 为 TGS 流程，`2696/2698/2700` 为 AP 与 RPC 交互。随后按上一阶段相同的 TCP、DCE/RPC 和 GSSAPI 重组方法处理 `tcp.stream == 10`。

恢复出的第三个任务名是 `\dNnouHfT`，动作读取 `C:\flag.jpg` 并把文件 Base64 写入描述。脚本中虽出现 AES 辅助密钥 `Ozunm03CgPP5P4BNFhroAQ==`，实际文件传输分支仍只做 Base64，因此读取任务响应的 `Description` 后直接解码即可得到 JPEG。

以管理员口令提取图片隐写内容：

```bash
steghide extract -sf flag.jpg -p 'Taylor@1989'
```

得到的 `flag.txt` 是一段 Base64 编码的 `IV || AES-CBC ciphertext`，还缺少最终密钥。PCAP 中 `frame 775` 和 `789` 是 Microsoft NTP 扩展的服务端响应：域控使用机器账户 NT Hash 为 48 字节 NTP body 计算 16 字节 MD5 MAC，并附带机器账户 RID。按 Hashcat 31300 格式导出：

```text
$sntp-ms$cb1877ec7aeeffb785f5689e483f0a3b$1c0111e900000000000a4c034c4f434ced54e820c41a9b8ce1b8428bffbfcd0aed554c56e832914ced554c56e833a7cd
$sntp-ms$8e8bab42e2cac7e5ef5d252f1eb63a5b$1c0111e900000000000a4c274c4f434ced54e820c5fea811e1b8428bffbfcd0aed554c868a16e29aed554c868a176f88
```

```bash
hashcat -m 31300 timeroast.txt /usr/share/wordlists/rockyou.txt
```

第二条材料破解为机器账户口令 `*joker*123`。最终 AES key 是该口令的 SHA-256，密文前 16 字节为 IV：

```python
import base64, hashlib
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

raw = base64.b64decode(open('flag.txt', encoding='ascii').read())
key = hashlib.sha256(b'*joker*123').digest()
print(unpad(AES.new(key, AES.MODE_CBC, raw[:16]).decrypt(raw[16:]), 16).decode())
```

最终得到：

```text
SUCTF{7h47_71m3_1_Cr4ck3d_3v3ry_3ncryp710n_1n_7h3_C7F_45_4_G3n1u5_H4ck3r_0nly_70_F1nd_7h3_Ul71m473_H1dd3n_Fl4g_15_7h3_P33rl355_B34u7y_K4nn0_5370!!!!}
```

TimeRoast 的补充资料可参考 [The Hacker Recipes 的技术说明](https://www.thehacker.recipes/ad/movement/kerberos/timeroast)。正文已经给出本题所需的协议结构、提取格式和解密关系，不依赖该外链也能复现。

## 方法总结

本题的主线是“凭据逐层升级，计划任务逐层取证”：先从 NetNTLMv2 得到 `kanna.seto` 口令，解开 NTLM 密封的 TSCH 流；再从 Kerberos AP-REP 恢复 GSSAPI subkey，解开第二条 TSCH 流并取得管理员 PKINIT 材料；随后利用 U2U 票据中的 `PAC_CREDENTIAL_INFO` 恢复管理员 NT Hash；最后解开管理员 TSCH 流，并用 TimeRoast 找到机器账户口令作为最终 AES 密钥。

最容易失败的地方有四个：不能把 TCP 帧等同于 DCE/RPC fragment；必须区分 GSSAPI 请求和响应的 key usage；必须区分 TGT session key、AP-REP subkey 与 PKINIT AS-REP key；看到脚本里定义 AES 函数也不能据此断言文件一定经过 AES，仍要追踪真实调用路径。把每一层的“身份、密钥、usage、载荷方向”单独记录，远比堆叠一个数百行的一次性脚本更可靠。
