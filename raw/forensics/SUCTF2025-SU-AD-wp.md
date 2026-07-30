# SU_AD

## 题目简述

题目由三份 Windows 域环境抓包组成，依次记录 SharpADWS 强制改密、Kerberos 认证的 PsExec/SMB3 会话，以及 wmiexec-Pro 通过 WMI 事件订阅执行混淆 VBS 的过程。解题目标是从认证握手恢复凭据和会话密钥，逐层解密 ADWS、Kerberos、SMB3 与 DCE/RPC 流量，最后导出并解开 `flag.zip`。

仓库没有收录原始 PCAP，只保留了官方题解、命令输出和 20 张过程截图。因此下述协议链和关键值可以由官方证据相互核对，但无法仅凭当前仓库重新跑一遍抓包解析。

## 解题过程

### 1. 从 NetNTLMv2 恢复 `sk` 的密码

第一份 `adws-1.pcapng` 中，SharpADWS 通过 ADWS/NMF 通信，并使用 NetNTLMv2 认证。先分别提取 Type 3 响应和 Type 2 Server Challenge：

```bash
tshark -n -r adws-1.pcapng \
  -Y 'ntlmssp.messagetype == 0x00000003' \
  -T fields \
  -e ntlmssp.auth.username \
  -e ntlmssp.auth.domain \
  -e ntlmssp.ntlmv2_response.ntproofstr \
  -e ntlmssp.auth.ntresponse

tshark -n -r adws-1.pcapng \
  -Y 'ntlmssp.messagetype == 0x00000002' \
  -T fields \
  -e ntlmssp.ntlmserverchallenge
```

抓包中可见用户 `sk`、域 `sk.com`，两次 Server Challenge 为：

```text
a780092344838bb2
b7325db726cdddbe
```

第一条响应的 NTProofStr 为 `0233f3f42302daa392aee9bb52a6887c`。把数据整理为 Hashcat 5600 模式要求的

```text
USER::DOMAIN:SERVER_CHALLENGE:NT_PROOF:BLOB
```

再使用字典恢复密码：

```bash
hashcat -a 0 -m 5600 ntlm_hash.txt rockyou.txt
```

结果为：

```text
sk / Eminem01
NT hash: 97b536419f4e41689f8d4f45523597a2
```

将该 NT hash 作为 RC4-HMAC（Kerberos enctype 23）密钥写入 keytab，并在 Wireshark 的 Kerberos 首选项中加载 keytab，即可解开 NMF 内的 ADWS 请求。明文请求修改了目标账户的 `unicodePwd`；这不是普通 LDAP 查询，而是一次强制改密操作。恢复出的管理员新密码为：

```text
1202)78M5CcE=+!2
```

### 2. 解开 Kerberos 认证的 SMB3

第二份 `psexec-kerberos-2.pcapng` 的 SMB 会话使用 Kerberos，而非 NTLM。先用管理员密码和 Kerberos salt `SK.COMAdministrator` 推导 AES-256 长期密钥：

```python
from impacket.krb5.crypto import _enctype_table

key = _enctype_table[18].string_to_key(
    "1202)78M5CcE=+!2",
    b"SK.COMAdministrator",
    None,
)
print(key.contents.hex())
```

得到：

```text
42711a4835ddf78bca7d46ca4a9aae640df23cb34e4bcb39c3bb23b699bddf8c
```

把该 AES-256 密钥加入 keytab 后，Wireshark 能解密 AS/TGS 交换，并从服务票据中得到 SMB 使用的 Service Session Key：

```text
057ca236576c77a46c3974840f1a407f
```

[微软对 SMB2/SMB3 密钥的说明](https://learn.microsoft.com/en-us/archive/blogs/openspecification/smb-2-and-smb-3-security-in-windows-10-the-anatomy-of-signing-and-cryptographic-keys)指出，Kerberos 与 NTLM 认证取得会话密钥的路径不同；这里不能照搬 NTLM 自动解密流程。官方截图中目标 SMB Transform Header 的 Session ID 为 `0x0000140010000021`，在 Wireshark 的 SMB2 “Secret session keys for decryption” 表中需按字节序写成：

```text
Session ID: 2100001000140000
Session Key: 057ca236576c77a46c3974840f1a407f
```

添加后 SMB3 负载被正确解密。从“导出 SMB 对象”列表可以取出：

```text
\WKHRGvOA.exe
\RemCom_communication
\flag.zip
```

其中 `flag.zip` 是最后需要解密的归档。

### 3. 还原 wmiexec-Pro 的混淆 VBS

第三份 `wmiexec-pro-3.pcapng` 记录的不是传统的明文命令行。wmiexec-Pro 通过 WMI 永久事件订阅把 VBS 数据写入 WMI 类，再经 DCE/RPC 远程执行。加载前面生成的 keytab 后，可以在解密后的 GSS-Krb5/DCE-RPC stub 中看到一大段由算术表达式组成的字符串。

混淆器的规则并不复杂：

1. 分隔符由 `chr(412650 / 9825)` 计算得到，即 `*`；
2. 每个分段都是只含整数和 `+`、`-`、`/` 的算术表达式；
3. 对每段求值并转为字符，按顺序拼接即可恢复原始 VBS。

可用下面的核心逻辑还原：

```python
encoded = "1322-1254*-2839+2944*5205-5096*..."
delimiter = chr(int(412650 / 9825))
decoded = "".join(chr(int(eval(item))) for item in encoded.split(delimiter))
print(decoded)
```

这里的 `encoded` 应替换为解密后的完整 stub 内容；不要直接复制运行未知 VBS。恢复出的脚本还包含一段 Base64 字符串：

```text
QzpcV2luZG93c1w3ei5leGUgeCAtcG9PQ0RMYmtaOU10dTY3QWx5aDh1QWFGSHk2S0RzQ2JHIC15IEM6XFdpbmRvd3NcZmxhZy56aXA=
```

Base64 解码后得到原始命令：

```text
C:\Windows\7z.exe x -poOCDLbkZ9Mtu67Alyh8uAaFHy6KDsCbG -y C:\Windows\flag.zip
```

因此压缩包密码是：

```text
oOCDLbkZ9Mtu67Alyh8uAaFHy6KDsCbG
```

用该密码解开先前导出的 `flag.zip`，得到：

```text
flag{Sk_l!kEs_aD_BuT_Ad_i5_7Oo_Hard_T_T}
```

## 方法总结

本题的决定性障碍是从三份既有流量中恢复并串联证据，因此归入 Forensics，而不是把题目仅按 AD 横向移动归入 Pentest。完整链路为：NetNTLMv2 握手恢复弱口令与 NT hash，keytab 解 ADWS 强制改密，管理员密码派生 Kerberos AES key，服务票据暴露 SMB Session Key，手动关联 Session ID 解 SMB3，最后从 DCE/RPC stub 还原 VBS 和归档密码。

处理这类题时应记录每一层的输入与输出，尤其是账户、realm/salt、enctype、Session ID、Session Key 和导出对象名；只在 Wireshark 中“点到明文”而不保存这些关联值，很难稳定复现整条证据链。
