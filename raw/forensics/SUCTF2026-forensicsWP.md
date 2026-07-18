# SUCTF2026-forensics

## 题目简述

题目给出一份由 FTK Imager 生成的 Windows 系统盘 AD1 镜像，需要从注册表、Windows 记事本状态文件、uTools 剪贴板、Cherry Studio / Ollama 数据库和应用日志中回答 7 个问题。最终提交格式为：

```text
SUCTF{MD5(答案1_答案2_答案3_答案4_答案5_答案6_答案7)}
```

其中密钥并未完整保存在单一位置：记事本残留只给出“四段密钥、按 1-4-3-2 重排、其中一段由 AI 生成”等规则，四段内容需要跨多个应用痕迹恢复。

## 解题过程

先用 FTK Imager 挂载或导出 AD1 镜像，再按问题定位对应 artifact。时间统一换算为 UTC+8，字符串的大小写、标点和换行字节必须保持原样。

### 1. 最后关机时间

导出 `C:\Windows\System32\config\SYSTEM` 注册表配置单元，使用注册表取证工具查看系统关机记录。最后一条完整关机记录显示系统于 `2026-03-05 16:46:12` 启动，并在 `2026-03-05 17:23:06` 关闭，因此答案为：

```text
2026/03/05T17:23:06
```

![系统关机记录](SUCTF2026-forensicsWP/suforensics.png)

### 2. 记事本删除内容的 MD5

新版 Windows 记事本会把未保存标签页保存在：

```text
%LOCALAPPDATA%\Packages\Microsoft.WindowsNotepad_8wekyb3d8bbwe\LocalState\TabState\
```

本镜像中的关键文件为 `992ff4a3-c3e9-401e-9320-82ddc5fa9d31.bin`。这类 TabState 文件保存文本缓冲区及编辑状态，即使界面上的文字已被删除，旧缓冲区仍可能残留。可按 [Notepad State Library](https://github.com/ogmini/Notepad-State-Library) 所实现的格式解析并重组缓冲区；该项目的关键价值是给出了 Windows 11 Notepad 状态文件的结构和恢复代码，因此不依赖网页正文也能完成解析。

恢复出的内容为：

```text
Key instructions:
1.Key must not be entirely stored on disk.
2.The key has four parts
3.The Key requires reshuffling order:1-4-3-2
4.There is a key generted by AI
complete
```

计算哈希时必须保留恢复结果中的 `0x0d` 换行字节，不能擅自规范化为 `0x0a`。得到：

```text
c1c4c50f51afc97a58385457af43e169
```

### 3. 第一密钥

uTools 剪贴板数据位于 `%APPDATA%\uTools\clipboard-data\`。官方 WP 参考的 [uTools 剪贴板取证文章](https://www.cnblogs.com/WXjzc/p/18129696) 的有效信息是：剪贴板正文并非明文存储，需要从本机 uTools 程序中恢复密钥并对数据文件解密；真正决定本题参数的仍是镜像内的程序代码。

`%APPDATA%\uTools\_asar_main_tmp\main.js` 表明剪贴板使用 AES-256-CBC，IV 固定为 `UTOOLS0123456789`，密钥来自原生模块的 `getLocalSecretKey()`。逆向 `win32-x64.node` 可还原其派生过程：读取 `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`，把机器 GUID 夹在两段静态字符串之间，再取 MD5 的十六进制文本。

```text
prefix      = H22`OB~6i{A{TXqIqPEg
MachineGuid = dfa96070-797f-4b50-bb3e-d478d5c44179
suffix      = |Wea6ywQQ`1q>_QyY2f1
AES key     = 5a569f2670ad9d9765df113e1417083f
AES IV      = UTOOLS0123456789
```

解密各时间目录后，`1772701170720` 中出现“第三密钥为第二密钥生成时间的时间戳”；继续检查时间异常的旧目录 `1772700955558`，恢复第一密钥：

```text
zQt$d3!GIS9l.aR@7ELN
```

这里 `9l` 的第二个字符是小写字母 `l`，不能按截图字形误抄为数字 `1`。

![uTools 中恢复出的第一密钥](SUCTF2026-forensicsWP/suforensics-6.png)

### 4. 第二密钥的对话 ID 与时间

Cherry Studio 的 IndexedDB 位于：

```text
%APPDATA%\CherryStudio\IndexedDB\file__0.indexeddb.leveldb\
```

不能只查看当前数据库视图；还要解析 LevelDB 的 `000003.log`，按记录键重组会话、消息和 message block。恢复出的“第二密钥生成尝试”会话中，用户确认采用 AI 生成的候选密钥；会话 ID 为 `019cbe60-6803-70fe-8ab5-e0035399980f`，关键消息的 `updated_at` 为 `2026-03-05 22:25:24.2129715+08:00`。按题目要求截断到秒并以下划线连接：

```text
019cbe60-6803-70fe-8ab5-e0035399980f_2026/03/05T22:25:24
```

会话同时给出第二密钥及其生成方式：先执行 `openssl rand -base64 32`，再把 `+/` 换成 `-_` 并删除 `=`，生成 URL-safe 字符串。本次实际采用的第二密钥为：

```text
4dE23eFgH7kLmNpOqRstUvWxYz012345678901234567890123456789
```

把生成时间 `2026-03-05 22:25:24+08:00` 转为 Unix 时间戳，得到第三密钥：

```text
1772720724
```

### 5. 完整密钥

继续检查 `%APPDATA%\uTools\database\` 下的 LevelDB，可恢复记录：

```text
key4:A9!fK2@pL4#tM6$wN8%yR1^uD3&hJ5*Z
```

这就是第四密钥。按记事本指示的 `1-4-3-2` 顺序直接拼接，段间不加连字符或其它分隔符：

```text
zQt$d3!GIS9l.aR@7ELNA9!fK2@pL4#tM6$wN8%yR1^uD3&hJ5*Z17727207244dE23eFgH7kLmNpOqRstUvWxYz012345678901234567890123456789
```

注意第四段末尾是大写 `Z`。

### 6. Ollama 的 `no such host` 时间

在 `%LOCALAPPDATA%\Ollama\app.log` 中搜索 `no such host`，命中日志：

```text
time=2026-03-05T21:58:17.244+08:00 ... lookup ollama.com: no such host
```

按题目格式取到秒：

```text
2026/03/05T21:58:17
```

![Ollama 日志中的 no such host](SUCTF2026-forensicsWP/suforensics-9.png)

### 7. 获得固定格式提示词的 message ID

回到 Cherry Studio 的 `000003.log`，用上一问附近出现的 OpenSSL 命令定位相应 message block。恢复记录表明，助手消息给出了“生成 256 位 Base64 密钥”和“转换为 URL-safe 版本”的完整命令；该 block 对应的 `messageId` 为：

```text
40854344-3f6e-4464-a07f-b39d42f5adc5
```

![Cherry Studio 恢复记录中的 messageId](SUCTF2026-forensicsWP/suforensics-10.png)

### 8. 计算 flag

将上述 7 个答案严格按题号排列，以 `_` 连接。第四个答案内部本身也含一个下划线，因此不能再拆项；对最终 306 字符的 UTF-8 字符串计算 MD5，结果为：

```text
39e850db5d740c54df4281e39fb3866d
```

最终 flag：

```text
SUCTF{39e850db5d740c54df4281e39fb3866d}
```

![七项答案拼接后的 MD5](SUCTF2026-forensicsWP/suforensics-11.png)

## 方法总结

这题的关键不是盲目全文搜索，而是把每个问题映射到正确的 Windows artifact：关机时间看 SYSTEM hive，删除文本看 Notepad TabState，剪贴板看 uTools 的加密数据与机器绑定密钥，AI 对话看 Cherry Studio IndexedDB / LevelDB，客户端报错看 Ollama 日志。

处理应用数据时应同时保留“存储位置、编码或加密机制、时间字段和记录关联键”。尤其要警惕三个易错点：TabState 哈希前不能改写换行字节；LevelDB 当前值不等于完整历史，需要恢复日志记录；最终字符串中的 `l/1`、`Z/z`、内部下划线都必须逐字核对。完成后再按题面公式独立重算摘要，才能确认整条证据链没有抄写错误。
