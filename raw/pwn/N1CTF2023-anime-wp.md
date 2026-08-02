# N1CTF 2023 anime Writeup

## 题目简述

题目把 Lua 5.4.6 静态链接进一个 CGI 程序。Nginx 会把以 `.lua` 结尾的路径交给 `/opt/cgi` 执行，而上传脚本根据查询串保存文件：

```lua
local fname = cgi.querystr() .. ".jpg"
if #fname > 5 and #fname < 32 then
    print(cgi.saveto(fname))
end
```

附件中的 `exploit.lua` 给出了高层利用链，但 `addrof`、`write_tbl` 和 `read_add16` 只是带有魔数的占位函数；真正可执行的 `pwn.out` 是手工替换过指令的 Lua 字节码块。解题需要先绕过上传后缀，再理解被篡改字节码如何破坏 Lua VM 的类型不变量，最终在静态 PIE 的 CGI 进程中构造 ROP 和原生 shellcode。

## 解题过程

### 用 NUL 截断上传可执行字节码

官方上传命令把查询串设为 `pwn.lua%00`。Lua 字符串本身可以包含 NUL，所以长度检查看到的是 `pwn.lua\0.jpg`；`cgi.saveto` 最终把它传给 C 文件接口，文件名在 NUL 处截断，实际保存为 `pwn.lua`。随后访问 `/data/pwn.lua`，Nginx 就会让 CGI 加载这个二进制 Lua chunk。

### 对照正常字节码定位三个伪造原语

将官方 `exploit.lua` 用原版 Lua 5.4.6 编译，再与题目提供的 chunk 反汇编结果逐函数比较，可以看到三个函数的普通指令被替换：

- `addrof(x)` 使用 `OP_FORLOOP`。该指令默认寄存器已经是数值型 `TValue`，内部直接读取 payload；把任意 Lua 对象放入对应寄存器后，它会把对象指针当整数更新并返回，从而泄漏对象或轻量 C 函数地址。
- `read_add16(a)` 先创建一个长字符串，再用两次 `OP_FORLOOP` 只修改其 payload 而保留字符串类型标签。随后 `OP_LEN` 把攻击者给出的地址当成 `TString *`，读取结构偏移 `+0x10` 的长度字段。调用 `read_add16(a-0x10)` 即可读取地址 $a$ 的 8 字节数据。
- `write_tbl(a,v)` 的首条指令被替换为 `OP_SETLIST`。传入的“表”其实是一段用 `string.pack` 布置的伪造 `Table`，令数组指针指向目标地址，`SETLIST` 便会把 `TValue v` 写入该地址。该写入同时覆盖相邻类型标签，因此官方脚本每次都按已知布局安排写入顺序。

伪造写表的核心形态是：

```lua
local fake = string.pack("I4j", 256, target)
write_tbl(addrof(fake) + 24 - 12, value)
```

### 泄漏基址并覆盖返回地址

`tostring` 在全局表中是轻量 C 函数，其 `TValue` payload 直接保存原生函数地址。附件对应 CGI 二进制中：

```lua
base = addrof(tostring) - 0x2ab50
environ_ptr = read(base + 0x52cd8)
stack = read(environ_ptr)
```

从 `environ` 得到栈地址后，向低地址扫描，寻找静态 PIE 启动代码 `base+0x3bb5` 的返回标记；目标返回地址位于该标记前 48 字节。官方脚本再用任意写依次布置：

```text
pop rdi ; ret      base + 0x3127
shellcode_address
pop rsi ; ret      base + 0x62b9
0x1000
pop rdx ; ret      base + 0x3825f
7
mprotect           base + 0x3d11f
shellcode_address
```

shellcode 放在 Lua 长字符串的数据区，即 `addrof(shellcode)+24`。静态 libc 的 `__mprotect` 包装函数会在发起系统调用前把起始地址向下对齐并扩展长度，所以这里可以直接传入未页对齐的字符串地址。shellcode 先创建管道，把 `getflag!` 写入预期的写端，再将读端复制到标准输入并执行 `/getflag`。环境中的 `/getflag` 是 SUID 程序，正确口令使其读取仅 root 可访问的 `/flag`，输出沿 CGI 标准输出返回。

附件环境中的结果为：

```text
n1ctf{anime_1s_all_y0u_need}
```

## 方法总结

这条利用链跨越了三个边界：Lua 字符串与 C 文件名对 NUL 的语义差异负责上传；被人为替换的 VM 指令破坏 `TValue` 类型约束，提供地址泄漏和任意读写；最后利用静态 PIE 中的固定相对偏移完成栈定位、`mprotect` 和 shellcode 执行。面对自定义或篡改字节码题，最有效的办法是用相同解释器版本生成基准 chunk，逐条做差，而不是仅从高层 Lua 源码猜测行为。
