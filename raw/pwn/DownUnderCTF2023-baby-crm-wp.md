# DownUnderCTF 2023 baby crm Writeup

## 题目简述

这是一个 C++ CRM 程序。`Order` 的两个构造函数都没有初始化 `description_`，但析构函数无条件执行 `delete description_`。结合不同菜单函数复用同一栈槽，可以释放仍被全局客户数组引用的 `Customer`，再把该堆块复用为可编辑的订单描述，形成 UAF 与对象类型混淆。

## 解题过程

先创建客户 0 和客户 1。调用一次 `alter_customer(0)`，让局部变量在栈上留下客户 0 指针；紧接着进入 `help()` 并选择 Order help。该函数构造局部 `Order o`，其未初始化的 `description_` 恰好复用先前栈槽，析构时把客户 0 的堆块释放，但 `customers[0]` 仍保存原地址。

`Customer` 对象和 `malloc(0x50)` 的订单描述落入相同 glibc chunk 大小。给客户 1 添加订单后，描述缓冲区会复用刚释放的客户 0，因此：

```text
customers[0] ---------+
                      v
                 freed Customer chunk
                      ^
Order.description ----+
```

订单描述为空时，旧对象内容没有被覆盖。显示客户 1 会固定输出 0x50 字节描述，从中泄漏堆指针。之后编辑这段描述，就等于伪造 `customers[0]` 指向的 C++ 对象。

libstdc++ 的 `std::string` 对象开头包含数据指针、长度和容量。将客户 0 的 `name_` 伪造成“指向任意地址、长度 8、容量很大”的字符串，即可构造读写原语：

```python
def fake_name(address, preserved_tail):
    return p64(address) + p64(8) + p64(0x4141414141414141) + preserved_tail

def read64(address):
    edit_order(1, 0, fake_name(address, tail))
    show_customer(0)
    return u64(io.recvn(8))

def write64(address, value):
    edit_order(1, 0, fake_name(address, tail))
    change_customer_name(0, p64(value))
```

利用堆中的分配指针和题目附带的 libc 确定 libc 基址，读取 `libc.environ` 得到栈地址。再把 `execve("/bin/sh",0,0)` 的 ROP 链分 8 字节写到主循环保存返回地址附近，选择退出菜单触发：

```text
DUCTF{0u7_0f_5c0p3_0u7_0f_m1nd}
```

## 方法总结

核心缺陷是未初始化拥有所有权的指针。局部栈残留把析构转化为错误释放，堆大小匹配又让描述缓冲区覆盖悬空的 `Customer`。伪造 `std::string` 后可把正常的打印和赋值接口升级为任意读写。构造函数必须把 `description_` 初始化为 `nullptr`，并使用一致的 RAII 容器管理内存。
