# Bash Injection

## 题目简述

WelcomeCTF2021 的 Bash Injection 是一个命令行登录服务。程序读取用户名和密码后，把二者直接插入字符串，再通过 `bash -c` 执行 `login.sh`。目标是打破引号边界并读取脚本中的 flag。

## 解题过程

`run.sh` 的关键逻辑是：

```bash
read user
read pass
bash -c "./login.sh \"$user\" \"$pass\""
```

虽然变量在最终命令中看似位于双引号内，但整条命令被重新交给 `bash -c` 解析。输入中的引号、分号和注释符会在第二次解析时重新获得语法含义。

把用户名设为：

```text
"; cat login.sh; #
```

密码可填写任意值。拼接后的命令可理解为：先结束原有参数，执行 `cat login.sh`，再用 `#` 注释掉剩余内容。`login.sh` 源码中直接包含成功登录时输出的 flag：

```text
greyhats{86sh_1n73ct10n_y6333}
```

也可以把 `cat login.sh` 换成其他只读命令验证注入，但解题所需信息在脚本本身已经完整存在。

## 方法总结

漏洞不是普通的凭据绕过，而是将不可信输入重新送入 shell 解释器。参数化调用 `./login.sh "$user" "$pass"` 本身可以安全传值，外层 `bash -c` 却把数据重新变成代码。防御方式是删除二次解释层，并把用户输入作为独立参数传递。
