# TSGCTF2021 Chat WP

## 题目简述

题目实现了一套通过两个命名管道通信的 C++ 聊天服务。消息保存在：

```cpp
std::variant<IntData, StringData> data {0xcafe};
```

房间原本只允许一个 host 和一个 client；`Client` 析构函数负责删除 `h2c`、`c2h` 两个 FIFO。利用链先让进程异常终止而不执行析构，从而反复重连同一管道，再把重连时的用户名当成协议行注入，最终触发堆泄露、堆溢出和 tcache poisoning。

## 解题过程

### 1. 让 `std::variant` 进入 valueless 状态

先把 `data` 切换为 `StringData`，再选择整数类型并提交超出 `unsigned long long` 的纯数字：

```text
100000000000000000000
```

字符检查只确认每个字节都是数字，随后 `stoull` 抛出异常。转换赋值发生在 `std::variant` 从 `StringData` 切换到 `IntData` 的过程中，因此异常后 variant 可能处于 `valueless_by_exception` 状态。菜单层捕获了这次异常，但之后选择发送消息会执行：

```cpp
std::visit(visitor, data);
```

它抛出未捕获的 `std::bad_variant_access`，运行库调用 `terminate`。题目环境不会在该路径上展开栈，因此 `Client::~Client()` 没有执行，命名管道也没有删除。进程虽死，另一端仍可保留连接状态，新进程可以重连同一房间。

### 2. 把重连用户名变成伪造数据包

初始化连接时，host 会把自己的名字直接写入 FIFO。保留一个 client 后，反复重连 host，就能依次注入协议所需的三行：

```text
2             # T_STR
1088          # 声明长度
QQ==          # 实际只解码出一个字节 "A"
```

接收端按声明长度分配 `length+1` 字节，但 `StringData` 仍把成员 `length` 保存为 1088。随后再次发送该对象时：

```cpp
Base64encode(buf, str, length);
```

会把只有首字节初始化的整段堆内存编码出去。借此读出 unsorted-bin/main-arena 指针，官方环境按：

```python
libc_base = leaked_pointer - 0x1ebfe0
```

计算 libc 基址。

### 3. 用长度与 Base64 内容不一致造成堆溢出

解码构造函数同样信任声明长度：

```cpp
char *buf = (char *)malloc(length + 1);
Base64decode(buf, b64_buf);
buf[length] = 0;
```

它没有确认 Base64 实际解码长度不超过 `length`。通过重连依次发送类型 `2`、较小长度 `88`，再发送能解码为 `0x70` 个 `A` 加一个指针的长 Base64 数据，可以越过目标堆块并改写相邻 tcache 元数据。

在若干 80、1、160 字节字符串分配与异常退出组成的堆整理后，把溢出的目标指针设为：

```python
free_hook = libc_base + 0x1eeb28
system = libc_base + 0x55410
```

后续消息分配落到 `__free_hook`，再通过正常字符串传输写入 `system` 地址。

### 4. 以 variant 类型切换触发 `free("sh")`

让仍存活的 client 保存字符串 `sh`，然后把 variant 改成一个合法整数。类型切换会析构旧 `StringData` 并释放其字符缓冲区。此时：

```text
__free_hook == system
```

所以析构实际执行 `system("sh")`。把后续网络输入作为 shell 命令发送即可读取：

```text
TSGCTF{Jump_over_the_dtor_to_get_win}
```

## 方法总结

本题把 C++ 异常安全、资源清理和二进制协议长度错误连成一条利用链。variant 构造失败本身只造成异常，但未展开栈让 FIFO 跨进程残留，从而绕开房间互斥；协议又把声明长度分别用于分配、终止和重新编码，却不核对 Base64 实际长度，最终同时产生越界读和堆溢出。修复需要保证 RAII 清理在所有退出路径发生、显式处理 `valueless_by_exception`，并以实际解码长度作为唯一可信长度。
