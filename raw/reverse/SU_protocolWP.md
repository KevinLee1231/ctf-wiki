# SU_protocol

## 题目简述
题目是自实现 HTTP `/flag` 服务和自定义二进制协议。题干提示长度字段后一个字节必须为 `0x80`，末尾字节必须为 `0x16`，`you_input` 需要全部小写；程序实际只注册 `POST /flag`，其他请求不会进入核心逻辑。

HTTP body 不是直接明文，而是先经过一层 hex 解码，再按私有协议解析出 payload。payload 会被切成 13 个 8-byte block，反复调用解密/变换函数后与固定目标比较。最终答案不是服务端直接回显的字符串，而是构造能通过校验的 `you_input`，再按提示提交 `SUCTF{md5(you_input)}`。

## 解题过程
程序启动后只注册了一个真正的 HTTP 路由：POST /flag 。

0x140002124 开始的代码如下：

```
0x140002124: mov byte ptr [rbp - 0x18], 0xa
0x140002128: mov dword ptr [rbp - 0x17], 0x616c662f
0x14000212f: mov word ptr [rbp - 0x13], 0x67
0x140002135: lea rcx, [rbp - 0x30]
0x140002139: lea rdx, [rbp - 0x18]
0x14000213d: call 0x140004790
0x140002142: lea rcx, [rip + 0x597d7]
0x140002149: mov qword ptr [rbp - 0x60], rcx
0x14000214d: lea rcx, [rip - 0xc84]
0x140002154: mov qword ptr [rbp - 0x58], rcx
0x140002158: lea rsi, [rbp - 0x60]
0x14000215c: mov qword ptr [rbp - 0x40], rsi
0x140002160: mov rcx, rsi
0x140002163: mov rdx, rax
0x140002166: call 0x14003bab0
```

0x616c662f 按小端展开就是 /fla ，后面的 0x67 就是 g ，所以这里明确注册的是
/flag 。

实际行为也能验证这一点：

返回 404 • GET /flag
- POST /flag 才会进入业务逻辑

handle_flag 主逻辑：

handle_flag 在 0x1400014d0 。

核心代码如下：

```
0x1400014d0: push rsi
0x1400014d1: push rdi
0x1400014d2: push rbx
0x1400014d3: sub rsp, 0xd0
...
0x140001518: lea rcx, [rsp + 0x38]
0x14000151d: lea rdx, [rsp + 0x20]
0x140001522: call 0x140003e30
0x140001527: lea rcx, [rsp + 0x57]
0x14000152c: lea rdx, [rsp + 0x38]
0x140001531: call 0x140003a50
...
0x140001556: lea rdi, [rsp + 0x58]
0x14000155b: lea rbx, [rsp + 0xc0]
0x140001563: mov rcx, rdi
0x140001566: mov rdx, rbx
0x140001569: call 0x140001450
...
0x140001612: lea rcx, [rsp + 0xb8]
0x14000161a: mov rdx, rbx
0x14000161d: call 0x140001450
0x140001622: lea rdx, [rip + 0x51f05]
0x140001629: mov r8d, 0x68
0x14000162f: mov rcx, rdi
0x140001632: call 0x140052480
0x140001637: test eax, eax
0x140001639: je 0x14000168c
```

这里可以拆成三步：

1. 0x140003e30 先处理 HTTP body。

2. 0x140003a50 再做协议解析和 payload 提取。

3. 把 payload 切成 13 个 8-byte block，反复调用 0x140001450 解密，再和固定目标比较。

比较失败时返回 wrong input ，格式错时返回 invalid input ，比较成功时返回提示：

flag may be SUCTF{md5(you_input)}

第一层：HTTP body 不是直接协议，而是“协议字符串再 hex 一次”

0x140003e30 是第一层 hex 解码器。

```
0x140003e30: push r15
...
0x140003f8e: movzx r10d, byte ptr [rdi + r8]
0x140003f93: lea r9d, [r10 - 0x30]
0x140003f97: cmp r9b, 0xa
...
0x140003f9d: lea r9d, [r10 - 0x61]
0x140003fa1: cmp r9b, 5
...
0x140003fb4: lea r11d, [r10 - 0x30]
...
0x140003fbe: lea r11d, [r10 - 0x61]
```

这一段很典型，就是把 ASCII hex 还原成字节，并且明确接受的是小写字母 a-f 。

后面真正的协议入口在 0x14004fed0 / 0x14004ff30 ，它们都要求数据形如：

#<hex>\n

对应代码：

```
0x14004fed8: mov rcx, qword ptr [rdx]
0x14004fedb: mov rdx, qword ptr [rdx + 8]
...
0x14004fee7: cmp byte ptr [rcx], 0x23
0x14004feea: jne 0x14004ff05
0x14004feec: cmp byte ptr [rdx - 1], 0xa
0x14004fef0: jne 0x14004ff05
0x14004fef2: inc rcx
...
0x14004ff15: call 0x14004e9e0
```

所以 HTTP body 的真实格式不是：

60007c... 其实是：23363030303763...

也就是：("#" + inner_frame_hex + "\n").encode().hex()

第二层：协议帧结构

0x14004e9e0 和 0x14004f4b0 是真正的协议解析器。

其中 0x14004e9e0 负责：

- 对 #... 中的 hex 再解一次

- 检查帧头

- 抽出 payload

- 校验 checksum

关键位置如下：

```
0x14004f17c: cmp rsi, r13
0x14004f17f: je 0x14004f1f0
0x14004f181: sub r13, rsi
0x14004f184: cmp r13, 9
0x14004f188: jb 0x14004f1f0
0x14004f18a: cmp byte ptr [rsi], 0x60
0x14004f18d: jne 0x14004f1f0
0x14004f18f: movzx eax, word ptr [rsi + 1]
0x14004f193: rol ax, 8
0x14004f197: cmp ax, 3
0x14004f19b: jbe 0x14004f421
0x14004f1a1: movzx r14d, ax
0x14004f1a5: add r14d, -3
```

这里直接说明：

- 第 1 个字节必须是 0x60

- 第 2~3 字节是大端长度

- 实际 payload 长度是 length - 3

后面复制 payload 和做校验：

```
0x14004f2c0: movdqu xmm2, xmmword ptr [rsi + rcx + 6]
0x14004f2c6: movdqu xmm3, xmmword ptr [rsi + rcx + 0x16]
0x14004f2cc: movdqu xmmword ptr [rbx + rcx], xmm2
0x14004f2d1: movdqu xmmword ptr [rbx + rcx + 0x10], xmm3
...
0x14004f3d9: add dl, byte ptr [rsi + 3]

0x14004f3dc: add dl, byte ptr [rsi + 4]
0x14004f3df: add dl, byte ptr [rsi + 5]
0x14004f3e2: cmp dl, byte ptr [rsi + r13 - 2]
0x14004f3e7: je 0x14004f200
```

这里能看出协议的大致结构：

0x60 | len_hi len_lo | byte3 | byte4 | byte5 | payload... | checksum |
0x16

并且 payload 从 raw[6] 开始。

结合实际跑通后的帧，可以还原出成功分支吃的 inner frame 形状：

$$
60 00 7c 80 55 ?? <121-byte payload> <checksum> 16
$$

题目 hint 对应的就是：

- 长度字段后面那个字节是 0x80

- 协议最后一个字节是 0x16

第三层：只接受 type = 0x55 且 payload 长度为 0x79

0x140003a50 是协议类型分发。

```
0x140003a5c: lea rcx, [rsp + 0x40]
0x140003a61: call 0x14004fed0
0x140003a66: lea rcx, [rsp + 0x28]
0x140003a6b: mov rdx, rdi
0x140003a6e: call 0x14004ff30
...
0x140003ab8: movzx edx, byte ptr [rdx + 4]
0x140003abc: cmp edx, 0xf7
...
0x140003ac8: cmp edx, 0x21
0x140003ad1: cmp edx, 0x23
0x140003ada: cmp edx, 0x55
...
0x140003ae5: cmp dl, byte ptr [rax]
0x140003ae7: jno 0x140003c82
0x140003aed: sub rcx, rax
0x140003af0: cmp rcx, 0x79
0x140003af4: jne 0x140003d06
```

这里可以确认：

- 协议 type 在 raw[4]

真正接受的是 type == 0x55 • /flag

长度必须是 0x79 • payload

其他的 0x21 / 0x23 / 0xfb 分支虽然存在，但和 /flag 这题主线没有关系。

第四层：解密函数不是标准 TEA，要注意运行态 patch

解密函数在 0x140001450 。

```
0x140001450: push rsi
0x140001451: push rdi
0x140001452: push rbp
0x140001453: push rbx
0x140001454: mov eax, dword ptr [rcx]
0x140001456: mov r8d, dword ptr [rcx + 4]
0x14000145a: mov r9d, dword ptr [rdx]
0x14000145d: mov r10d, dword ptr [rdx + 4]
0x140001461: mov r11d, dword ptr [rdx + 8]
0x140001465: mov edx, dword ptr [rdx + 0xc]
0x140001468: mov esi, 0xc6ef3600
0x14000146d: mov edi, 0x20
...
0x140001496: sub r8d, ebx
...
0x1400014b3: sub eax, ebx
0x1400014b5: add esi, 0x61c88647
0x1400014bb: dec edi
0x1400014bd: jne 0x140001480
```

这题的坑在于：

- 盘上代码是 add esi, 0x61c88647

- 运行到不同环境时，这个立即数会被 patch

实际 dump 结果：
- powershell 运行态：0x61c88647
- cmd 运行态：0x61c88650

但是初始和并没有改，依然是：

$$
sum = 0xC6EF3600
$$

因此它不是标准 TEA 逆过程，不能直接套模板。

目标常量

成功路径最后比对的是一段固定字符串。

对应内存字符串为：
ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL_1FAEFB6177B4672DEE07F9D3AFC62588
CCD2631EDCF22E8CCC1FB35B501C9C86

程序的做法是：

- 从 payload 里取出前 104 字节

- 以最后 16 字节作为 key

- 按 8-byte block 做 13 次 0x140001450

- 结果和上面的目标比较

payload 的尾部 16 字节最终是：

7375323032362d6b6579736563726574

也就是 ASCII：su2026-keysecret

本地可以稳定打到成功提示的 payload 是：

```
802ba5e6806f7dd07b988241146e350f481ec220fe1536b67193671193ca08060fd065ddf9c197a
```

119d2f732d8c574e7fc8ca862a2a15e3e7312df0fe81b0f810bf27f7f8982b9a1880ac3d3fd128a
cabe866e82655cb2b536edf8714ec03162c91ed2c534c132a3347375323032362d6b65797365637

对应两条本地都能过的 inner frame：

```
60007c805500802ba5e6806f7dd07b988241146e350f481ec220fe1536b67193671193ca08060fd
```

065ddf9c197a119d2f732d8c574e7fc8ca862a2a15e3e7312df0fe81b0f810bf27f7f8982b9a188
0ac3d3fd128acabe866e82655cb2b536edf8714ec03162c91ed2c534c132a3347375323032362d6
b65797365637265744516

```
60007c805580802ba5e6806f7dd07b988241146e350f481ec220fe1536b67193671193ca08060fd
```

065ddf9c197a119d2f732d8c574e7fc8ca862a2a15e3e7312df0fe81b0f810bf27f7f8982b9a188
0ac3d3fd128acabe866e82655cb2b536edf8714ec03162c91ed2c534c132a3347375323032362d6
b6579736563726574c516

注意这里的 raw[5] 并不会影响 /flag 本地成功，所以本地样本存在歧义。

输入层次总结

本题一共至少有三层输入：

1. HTTP body：ASCII hex

2. #<inner_frame>\n

3. inner frame 内部的 payload

真正提交到服务端的是第 1 层。

也就是说，POST /flag 的 body 应该是：

$$
("#" + inner_frame_hex + "\n").encode().hex()
$$

本地验证脚本

```python
import hashlib
import urllib.request

FRAME_00 = (
```

"60007c805500802ba5e6806f7dd07b988241146e350f481ec220fe1536b67193671193ca08060f
d065ddf9"

"c197a119d2f732d8c574e7fc8ca862a2a15e3e7312df0fe81b0f810bf27f7f8982b9a1880ac3d3
fd128acabe"

"866e82655cb2b536edf8714ec03162c91ed2c534c132a3347375323032362d6b65797365637265
744516"

```python
)

def build_outer_body(frame_hex: str) -> bytes:
return ("#" + frame_hex + "\n").encode().hex().encode()

def post_flag(body: bytes) -> bytes:
req = urllib.request.Request("http://127.0.0.1:8080/flag", data=body,
method="POST")
with urllib.request.urlopen(req, timeout=3) as resp:
return resp.read()

def main() -> None:
outer = build_outer_body(FRAME_00)
result = post_flag(outer)
print(f"response = {result.decode()}")
print(f"outer_body = {outer.decode()}")
print(f"md5(outer_body) = {hashlib.md5(outer).hexdigest()}")

print(f"flag = SUCTF{{{hashlib.md5(outer).hexdigest()}}}")

if __name__ == "__main__":
main()
```

## 方法总结
- 核心技巧：自定义协议解析与块加密反解
- 识别信号：HTTP 层很薄，真正约束在 body parser 和固定 block 解密。
- 复用要点：先区分路由/格式错误/比较失败，再恢复 block 算法，反推出合法 payload。
