# 家目录里的秘密

## 题目简述

题目给出一份 Linux 用户家目录归档，要求从编辑器和同步工具遗留的配置中恢复两段 flag。关键工件分别是 VS Code 的本地文件历史，以及 `.config/rclone/rclone.conf` 中可逆混淆的密码字段。题目考查的不是连接配置里的示例 FTP 服务器，而是从已经取得的家目录证据中识别敏感信息残留。

## 解题过程

### VS Code 本地历史

解压家目录后，应同时搜索隐藏目录和被忽略文件。`ripgrep` 可以这样使用：

```bash
rg -F --hidden --no-ignore 'flag{' .
```

命中位置是：

```text
.config/Code/User/History/2f23f721/DUGV.c
```

其中保留了一行已经从原工作文件删除的注释：

```c
// flag{finding_everything_through_vscode_config_file_932rjdakd}
```

VS Code 默认的 Local History 会在每次保存文件时复制一个历史版本。Linux 下这类数据位于 `$HOME/.config/Code/`；Windows 和 macOS 对应 `%APPDATA%/Code/` 与 `$HOME/Library/Application Support/Code/`。即使项目未使用 Git、原文件已经删除，编辑器历史仍可能保存完整内容。

### Rclone 密码混淆

第二段线索位于：

```text
.config/rclone/rclone.conf
```

配置中的主机是 RFC 2606 保留的 `ftp.example.com`，因此无需尝试连接。真正的 flag 被放进 `pass` 字段，并经过 rclone 的 `obscure` 处理。这个机制必须可逆，因为 rclone 在没有额外密钥输入的情况下仍要恢复明文密码。

题目中的混淆值为：

```text
tqqTq4tmQRDZ0sT_leJr7-WtCiHVXSMrVN49dWELPH1uce-5DPiuDtjBUN3EI38zvewgN5JaZqAirNnLlsQ
```

rclone 源码的 `Reveal` 逻辑先做无填充 URL-safe Base64 解码，把前 16 字节作为 AES 的 IV，再用程序内置密钥解密剩余数据。无需自行重写算法，直接调用 rclone 的公开实现即可：

```go
package main

import (
	"fmt"

	"github.com/rclone/rclone/fs/config/obscure"
)

func main() {
	const encoded = "tqqTq4tmQRDZ0sT_leJr7-WtCiHVXSMrVN49dWELPH1uce-5DPiuDtjBUN3EI38zvewgN5JaZqAirNnLlsQ"
	fmt.Println(obscure.MustReveal(encoded))
}
```

在一个空目录中执行：

```bash
go mod init reveal-rclone
go get github.com/rclone/rclone/fs/config/obscure
go run .
```

输出即为第二段 flag。rclone 官方将该功能称为 [obscure](https://rclone.org/commands/rclone_obscure/) 而不是安全的密码加密；拿到配置文件的人可以按相同过程还原明文。

## 方法总结

- 核心技巧：搜索编辑器历史工件，并识别应用配置中的可逆密码混淆。
- 识别信号：家目录镜像中出现 `.config/Code/User/History`、`Backups` 或 rclone 的 `pass` 字段时，应把它们当成潜在敏感证据。
- 复用要点：取证搜索要包含隐藏目录；“看起来随机”不代表不可逆，若应用无需用户再次提供密钥就能使用某字段，应检查其公开解码实现。
