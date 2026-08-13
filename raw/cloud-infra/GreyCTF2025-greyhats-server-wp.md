# GreyCTF 2025 greyhats-server Writeup

## 题目简述

目标是一台使用 Ubuntu `authd` 和 Google OAuth broker 的 SSH 服务器。题目提供旧的本地 `student` 账号，但 `/flag` 仅 root 可读，SUID 程序 `/printflag` 的权限为 `0710`：普通用户甚至不能执行，root 组成员则可以执行。

服务安装的是 `authd 0.5.3`。该版本存在 CVE-2025-5689：一个此前未在 authd 数据库中出现的云用户首次通过 SSH 登录时，会在该 SSH 会话中被错误地视为 root 组成员。题目又把 broker 配成允许全部已认证用户，并允许任意 `@gmail.com` 账号通过 SSH 创建身份，因此可用自己的 Google 账号触发漏洞。

## 解题过程

先用题目提供的 `student` 凭据登录，检查系统和配置。构建脚本中的关键项为：

```text
authd version: 0.5.3
allowed_users = ALL
ssh_allowed_suffixes = @gmail.com
KbdInteractiveAuthentication yes
```

Canonical 的[安全公告](https://github.com/canonical/authd/security/advisories/GHSA-g8qw-mgjx-rwjr)说明：受影响版本低于 0.5.4；满足 SSH、broker、允许后缀和首次登录条件时，新 authd 用户会在当前 SSH 会话里获得 root 组访问能力，但不会直接获得完整 root 权限。这里的配置恰好满足全部前提。

退出旧账号后，以尚未在目标机登录过的个人 Gmail 地址作为 SSH 用户名，选择 Google broker，并按终端给出的设备授权流程在浏览器中完成 OAuth。登录成功后用 `id` 检查当前会话，可以看到错误附加的 root 组。于是权限为 `0710`、属组为 root 的 `/printflag` 已可执行。

`/printflag` 本身还存在一处 PATH 信任问题：

```c
FILE *fp = popen("date", "r");
```

程序没有使用 `/usr/bin/date` 的绝对路径。可在当前用户可写的主目录中放置名为 `date` 的可执行脚本，让它输出一个固定字符串，例如 `known-date`，再把该目录放在 PATH 最前。程序会删除 `date` 输出末尾的换行，而其 `fgets(stdin)` 不会主动删除输入换行，因此应使用 `printf` 发送同一字符串且不附带换行：

```bash
mkdir -p "$HOME/bin"
printf '#!/bin/sh\nprintf "known-date\\n"\n' > "$HOME/bin/date"
chmod +x "$HOME/bin/date"
printf 'known-date' | env PATH="$HOME/bin:/usr/bin:/bin" /printflag
```

即使 `/bin/sh` 在执行伪造的 `date` 时降低子进程权限，父进程 `/printflag` 仍保持 SUID root；比较通过后，它以有效 root 身份打开 `/flag`，输出：

```text
grey{wh0_kn3w_y0u_c4n_u5e_o4uth_f0r_ssh?!?_atFj5QNm3xgpctX8}
```

## 方法总结

完整利用链由三个边界错误组成：过宽的 OAuth 用户策略允许任意 Gmail 身份进入；`authd 0.5.3` 的首次 SSH 登录错误赋予 root 组；组执行权限又暴露了 SUID helper，而 helper 使用相对命令名调用 `date`。此题说明“没有直接 root”不等于影响有限：root 组可访问的程序或文件仍可能成为下一跳。修复应升级至 `authd 0.5.4` 或更高版本、收紧 `allowed_users` 与邮箱后缀，并在特权程序中使用绝对路径和安全环境。
