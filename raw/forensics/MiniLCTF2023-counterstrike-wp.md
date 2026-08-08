# MiniLCTF2023 - counterstrike

## 题目简述

附件只有 `victim.pcapng` 与 `victim's-server.pcap`，没有官方题解或服务源码。两份流量共同记录了一次“反制攻击”：先利用恶意 SVG/JAR 控制 Cobalt Strike 操作者的主机并窃取 Team Server 密钥，再用该私钥解密另一份 Beacon 流量，恢复被下载的加密 ZIP 及其键盘记录密码。

决定性证据都能在附件中闭环验证，不需要猜测出题人的环境。

## 解题过程

先检查服务端流量的 HTTP 对象：

```powershell
wsl tshark -r "victim's-server.pcap" -Y http.request `
  -T fields -e tcp.stream -e http.request.uri -e http.user_agent
```

可以看到 `Batik/1.7` 请求 `/evil.svg`，随后 Java 请求 `/EvilJar-1.0-jar-with-dependencies.jar`。SVG 内容把 JAR 作为 `application/java-archive` 脚本加载，并直接标注 `CVE-2022-39197`。对导出的 JAR 执行 `javap -c -p Exploit 'Exploit$1'`，可见 Windows 分支启动 PowerShell TCP 反向 shell，连接 `82.156.5.200:1033`。

跟踪该明文 TCP 流：

```powershell
wsl tshark -r "victim's-server.pcap" -q -z follow,tcp,ascii,8
```

攻击者依次执行 `whoami`、`ls`、`certutil -encode .cobaltstrike.beacon_keys keyb64` 和 `type keyb64`。输出是 1446 字节 Java 序列化 `KeyPair`。序列化数据中的私钥字段长度为 `0x27a`，从 PKCS#8 DER 头 `30 82 02 76` 开始截取 634 字节即可；`openssl rsa -inform DER -check` 会返回 `RSA key ok`。

另一份流量中，`/miniL1.html` 的 Dean Edwards packer 展开后等价于：

```javascript
function xorEncrypt(a, b) {
  let result = "";
  for (let i = 0; i < a.length; i++) {
    result += String.fromCharCode(a.charCodeAt(i) ^ b.charCodeAt(i % b.length));
  }
  return btoa(result);
}

// key: SECRET_KEY_balabalabala
// ciphertext: PiwtOwkXCw0+DHsHPi8OICEAFTEVHigYdhwlDCAXFAYYKjYoXC89
```

Base64 解码后用重复密钥 XOR，得到 flag 前半段：

```text
miniLCTF{U$e_CoB@ltStrIK3_wItH_CAuTI0N_
```

`/pOd9` 是 Beacon stage。配置显示 GET URI 为 `/IE9CompatViewList.xml`，提交 URI 为 `/submit.php`，元数据位于 `Cookie`，回包正文直接承载加密任务。用刚才提取的 RSA 私钥解开 Cookie 中的 PKCS#1 v1.5 元数据，可得到会话随机数：

```text
ab d6 5d 24 00 26 ab 1d 8f 0a 0e ca 97 f2 e7 10
```

对它计算 SHA-256，前后各 16 字节分别为：

```text
AES  = 25465e54b77dd74ef43712494d3fcaa0
HMAC = 8032472a398493c6bb29c08a1e54eac6
IV   = 6162636465666768696a6b6c6d6e6f70  # abcdefghijklmnop
```

POST 正文由若干 `4 字节大端长度 | AES-CBC 密文 | 16 字节 HMAC` 数据包组成。验证 HMAC 后解密，按 Beacon callback 类型重组，流量依次显示：

```text
COMMAND_FILE_LIST     C:\*
COMMAND_FILE_LIST     C:\flag\*
COMMAND_DOWNLOAD      C:\flag\flag.zip
CALLBACK_FILE_WRITE   186-byte encrypted ZIP
```

紧随其后的 `CALLBACK_KEYSTROKES` 数据把密码拆成四段：

```text
e83e449b-
2454-4093-b
50b-38b04
883e82b
```

重组得到 `e83e449b-2454-4093-b50b-38b04883e82b`。用它解密 ZIP 内的 `flag.txt`，得到：

```text
oR_6Et_pwNEd_1n_r3veRs3}
```

与前半段拼接，完整 flag 为：

```text
miniLCTF{U$e_CoB@ltStrIK3_wItH_CAuTI0N_oR_6Et_pwNEd_1n_r3veRs3}
```

## 方法总结

这类流量题需要按因果顺序还原，而不是只在 PCAP 中搜索 `flag`：恶意文档说明初始执行，反向 shell 泄露服务器私钥，私钥再解开 Beacon 元数据和会话密钥，最终才有能力重组文件与键盘记录。每个阶段都应保留可验证锚点——URI、JAR 字节码、DER 结构、Beacon ID、任务类型、文件长度和 ZIP 密码——这样即使没有官方 WP，也能形成完整且可复现的证据链。
