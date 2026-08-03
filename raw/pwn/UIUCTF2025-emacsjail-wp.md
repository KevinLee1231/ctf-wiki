# emacsjail

## 题目简述

服务读取一段 Emacs Lisp 对象表示，拒绝任何包含圆括号的输入，再用 `read-from-string` 解析第一个对象。对象必须满足 `functionp`，随后由 `funcall` 执行。

执行前，程序已用 `find-file-noselect "./flag.txt"` 把 flag 文件载入缓冲区，并对 `buffer-substring`、`insert-file-contents`、`call-process` 等危险函数符号添加 `:before` advice，一旦经普通函数调用进入这些符号就退出。漏洞来自两层语义差异：字节码函数的打印表示不含圆括号，而且部分 Emacs Lisp 原语有专用字节码 opcode，不经过被 advice 的函数符号。

## 解题过程

解释执行的 lambda 通常打印成：

```elisp
(lambda nil ...)
```

必然触发括号过滤。编译后的函数则可以由 reader syntax `#[...]` 表示，例如：

```elisp
#[0 "\300q\210ed{\207" ["flag.txt"] 2]
```

该对象由四部分组成：

- `0`：参数描述，表示无参数；
- 字节串：Emacs 字节码；
- `["flag.txt"]`：常量池；
- `2`：栈深度。

字节码对应的操作为：

```text
constant "flag.txt"
set-buffer
discard
point-min
point-max
buffer-substring
return
```

程序之前已经打开 `./flag.txt`，其缓冲区名称为 `flag.txt`。字节码先切换到该缓冲区，再取从 `point-min` 到 `point-max` 的全部内容。关键在于 `set-buffer` 和 `buffer-substring` 都有解释器专用 opcode；执行 opcode 时不会走 `advice-add` 挂在符号函数上的包装器。

输入中没有 `(` 或 `)`，`read-from-string` 仍能构造一个 `byte-code-function`，`functionp` 检查也会通过。`funcall` 的返回值最终由 `message` 输出：

```text
uiuctf{h3r3s_s0me_4dv1c3_d0nt_u53_emacs}
```

不同 Emacs 版本可能生成不同字节串，因此稳定做法是先在与题目相同版本中编译并用 `prin1-to-string` 查看表示；如果必须手写，则逐项核对 opcode、常量池和声明的栈深度。

## 方法总结

- 核心技巧：提交不含括号的 `byte-code-function` reader 对象，并利用专用字节码 opcode 绕过基于符号 advice 的函数拦截。
- 识别信号：过滤只检查源码字符串，运行时却接受已编译函数对象；安全层通过 monkey patch/advice 包装函数，但解释器可能有直达原语的 opcode。
- 复用要点：语言沙箱必须限制可接受的对象类型和执行能力，而不只是源代码字符。还要检查编译器、reader、字节码和原生 primitive 是否存在绕过高层 hook 的路径。
