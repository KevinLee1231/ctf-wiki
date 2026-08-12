# Wait a Minute

## 题目简述

服务表面上是一个 Python pyjail：输入先经过长度、黑名单、正则和 `ast.parse(..., mode="eval")`，最后才在空 builtin 的全局环境中 `eval`。真正的漏洞不在 `eval`，而在验证正则与外层 shell 超时处理的组合。

`server.py` 的 `base` 分支允许圆括号，`group_rd` 分支也允许一个由 `base` 组成的圆括号组；两者再被包在可重复的交替分支中。`run.sh` 用 `timeout 60 python server.py "$input"` 运行校验；只要子进程因超时退出，默认分支就把 `logs/err.log` 的内容回显。Dockerfile 在构建时把 `flag.txt` 复制为这个日志文件。

## 解题过程

### 找到可歧义的失败匹配

验证模式等价于：

```python
base = r'[a-zA-Z0-9=+\-/:_\."\'\s\(\)\[\]]*?'
group_rd = rf'\({base}\)'
pattern = re.compile(rf'^({base}|{group_rd}|\[{base}\])*$')
```

圆括号既可以被 `base` 消耗，也可以被 `group_rd` 消耗。对于大量 `()`，外层重复和两个候选分支存在大量不同分割方式。尾部再放入一个不属于允许字符集的 `{`，则整个匹配必定失败，但 Python 的回溯引擎会在发现失败前枚举前面圆括号的组合。`*?` 是惰性量词，不会消除这种交替分支造成的指数级回溯。

官方 solve 使用的输入为：

```python
payload = "()" * 30 + "{"
```

它长度为 61，低于 167 的限制，不包含黑名单单词，也不会到达 `ast.parse`；服务会卡在 `pattern.fullmatch(payload)`。

### 利用超时后的日志回显

包装器的核心逻辑是：

```sh
output=$(timeout "$TIME_LIMIT" python server.py "$input" 2>&1)
status=$?

case $status in
    0|1) echo "$output" ;;
    *) echo "Internal error (code $status). Report to admin: $(cat logs/err.log)" ;;
esac
```

GNU `timeout` 在 60 秒到期后返回非 0/1 的状态（通常为 124），从而进入最后一个分支。构建阶段又执行了：

```sh
cp /srv/app/*.txt /srv/app/logs/err.log
```

这里的 glob 包含 `flag.txt`，故 `logs/err.log` 实际保存的是 flag 内容。向服务提交上述单行 payload，等待超时提示，回显中的“Report to admin”字段即为 flag。

### 验证

该输入不会执行用户 Python 表达式；成功条件是服务在约 60 秒后返回内部错误和日志内容，而不是出现 `Result:`。仓库中给出的结果为：

```text
grey{9eT_i7_h0w_Y0u_1iv3_1t_10_t0E5_iN_wH3n_We_5t4nDin_0n_Bu5Ine5S}
```

## 方法总结

- 核心技巧：对带嵌套/交替量词的输入过滤器，验证“必败长串”是否诱发灾难性回溯；惰性量词不是 ReDoS 防护。
- 识别信号：pyjail 外层出现 shell `timeout`，同时非正常退出会回显日志、错误详情或诊断文件时，应把资源耗尽视为信息泄露原语。
- 复用要点：不要只检查 sandbox 的 `eval`、黑名单和 builtin。还要审计输入验证的复杂度，以及超时、异常和容器构建阶段是否把敏感文件接到了可回显的路径。
