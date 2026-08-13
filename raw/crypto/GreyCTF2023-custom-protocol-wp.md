# GreyCTF 2023 Custom Protocol

## 题目简述

服务通过 HTTP 接收空格分隔的“指令 参数”对，但会先对每个 token 做 Caesar 正向移位，再把结果当作真实命令。决定性障碍是恢复逐 token 变化的可逆编码，而不是 HTTP 漏洞，因此归入 crypto。先编码 `read ./main.go` 可读取服务源码，再调用源码中隐藏的取 flag 指令。

## 解题过程

第 $i$ 个 token 从 0 开始计数，服务使用的移位量为

$s_i=\operatorname{len}(token_i)+i$，

并对字母执行正向 Caesar 移位。要让服务最终看到目标 token，就应对目标做同样位数的反向移位；非字母字符保持不变。

对于第一组指令，目标是 `read ./main.go`：

```text
read     反向移 4  -> nawz
./main.go 反向移 10 -> ./cqyd.we
```

提交：

```text
nawz ./cqyd.we
```

即可读到 `main.go`。源码的操作码表除 `read`、`ping` 和 `print` 外，还有隐藏键 `giv3-fl4g-p1s`。对它按首 token 的长度反向移位，得到 `tvi3-sy4t-c1f`；其参数不会被使用，因此提交：

```text
tvi3-sy4t-c1f test
```

响应包含 flag。服务在返回前会把所有下划线替换为空格，所以应把花括号内部相应空格还原为下划线，最终为：

```text
grey{c43sar_g0t_4n_1nj3cti0n_f4618314c4d25169f5735ca0d4a29e41}
```

## 方法总结

这不是固定移位的普通 Caesar：位移同时依赖 token 长度和它在整个程序中的索引，指令与参数必须分别求逆。利用已知命令读源码，可以避免盲猜隐藏操作码；还要注意输出层的下划线替换，否则直接复制响应会得到错误 flag。
