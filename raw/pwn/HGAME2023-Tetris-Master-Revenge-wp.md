# Tetris Master Revenge

## 题目简述

Revenge 版本修复了直接按 `Ctrl+C` 返回 shell 的问题，并允许正常游玩累计到目标分数。游戏仍由 Bash 编写，其中玩家对“是否是俄罗斯方块大师”的回答会进入 `[[ ... -eq ... ]]`、`[[ ... -ne ... ]]` 等算术比较，这为命令注入留下了入口。

## 解题过程

游戏结束时的核心判断如下：

```bash
if [[ "$master" -eq "y" ]] && [[ "$score" -gt 50000 ]]; then
    echo -ne "cat /flag"
elif [[ "$master" -ne "y" ]] && [[ "$score" -gt "$target" ]]; then
    echo -ne "Keep Going"
else
    echo -ne "$score"
fi
```

第一种解法是正常或脚本化游玩。`score` 在重新开始游戏时不会清零，因此可以反复开局累计，超过 `50000` 分后触发读取 flag 的分支。

更直接的解法利用 Bash 算术求值。`-eq` 和 `-ne` 会把参数当作算术表达式处理；数组下标同样属于算术上下文，其中的命令替换会被执行。对大师询问输入：

```bash
arr[$(cat /flag)]
```

随后让本局游戏结束，使脚本进入上述比较。Bash 在解析数组下标时执行 `cat /flag`，读出的内容又成为非法算术表达式的一部分，因此会随错误信息出现在终端中。得到：

```text
hgame{Bash_Game^Also*Can#Rce^reVenge!!!!}
```

这里不能简单理解成字符串比较漏洞：决定性原因是 Bash 算术表达式会递归解析变量值和数组下标，并在该上下文中触发命令替换。GNU Bash 手册的 [Shell Arithmetic](https://www.gnu.org/software/bash/manual/bash.html#Shell-Arithmetic) 一节给出了这种求值语义；利用所需结论已在正文中说明。

## 方法总结

Bash 中 `[[ "$value" -eq 0 ]]` 并不等价于安全地检查一个已验证整数。只要 `value` 可控，算术求值就可能把变量名、数组下标和命令替换继续解释。修复时应先用严格正则验证十进制整数，再进行算术运算，并避免把原始用户输入放入算术上下文。
