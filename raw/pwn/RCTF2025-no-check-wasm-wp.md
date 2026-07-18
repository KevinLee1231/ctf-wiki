# no_check_WASM

## 题目简述

题目提供一份定制 V8 `d8`、基于 V8 提交 `42f5ff65d12f0ef9294fa7d3875feba938a81904` 的补丁，以及脚本上传服务。服务先要求求解 24 位 PoW：找到 6 个十六进制字符 `nonce`，使 `SHA-256("rctf" + nonce)` 等于给定哈希；随后接收不超过 102400 字节的 JavaScript，并在 60 秒限制内用下列命令运行：

```text
/home/ctf/d8 --no-memory-protection-keys <上传脚本>
```

`d8` 的 `read`、`write`、`os`、`load` 等辅助接口均被移除，只保留 `print`。flag 权限为 `0400`，容器另有 setuid 程序 `/read_the_flag`。因此目标是利用 V8 的 Wasm 验证缺陷获得原生代码执行，再启动 shell 或直接执行该程序。

## 解题过程

### 1. 定位真正的补丁漏洞

关键补丁位于 `src/wasm/function-body-decoder-impl.h`。V8 在函数结束、分支、循环等 merge point 调用 `TypeCheckStackAgainstMerge`，确认当前操作数栈至少含有目标 arity 所需的元素，并逐项检查类型。题目把 slow path 中的两段逻辑全部注释掉：

```cpp
// 检查 actual 与 arity
// 检查栈顶每个 Value 是否为 merge 声明类型的子类型
return true;
```

外层函数仍保留两个 fast path：arity 为 0 时直接成功；arity 为 1 且数量、类型恰好匹配时直接成功。其余情况都会进入已被掏空的 slow path。所以最方便的触发方式是让 merge arity 大于 1，或者故意让单返回值的实际类型与声明类型不同。

这不是 element section 或函数表校验漏洞。其直接后果是：模块可以声明若干返回值却不向栈压入任何值，也可以把 `i64` 当作 WasmGC 引用返回，验证和实例化仍然成功。

### 2. 构造 `i64` 与 `structref` 类型混淆

先定义一个仅含可写 `i64` 字段的 WasmGC struct，再创建一个声明为 `(i64) -> ref struct(i64)` 的函数，函数体却只返回输入的 `i64`：

```javascript
let structType = builder.addStruct([makeField(kWasmI64, true)]);
let castSig = makeSig([kWasmI64], [wasmRefType(structType)]);

let cast = builder.addFunction('from_i64', castSig)
  .addBody([kExprLocalGet, 0]);
```

正常 V8 会在函数末尾拒绝 `i64` 与 `structref` 的类型不匹配；补丁版本却接受它。把该结果交给 `struct.get 0`、`struct.set 0`，即可让 V8 把攻击者提供的 64 位整数当成对象地址，并读写该“对象”的首个字段：

```javascript
read64(addr)  = struct.get(cast(addr), 0)
write64(addr, value) = struct.set(cast(addr), 0, value)
```

exploit 中出现的 `addr - 7`、`addr + 1` 等修正来自该版本 V8 的对象标记与 struct 字段偏移，属于构建绑定常量，不能当作通用 Wasm 地址规则。

### 3. 泄露沙箱外的原生栈地址

仅有整数到引用的混淆还缺一个可用的原生指针。题目目录中的 exploit 利用“虚假多返回值”从未初始化槽位取出旧数据：

1. `nop` 声明返回三个 `i64`，实际函数体不压入任何值。
2. `foo` 声明接收三个 `i64`，并把前两个参数传给导入的 JavaScript 回调。
3. `leak` 先调用 `nop`，再立即调用 `foo`。验证器认为前者已经产生三个参数，运行时却会让后者消费残留的寄存器或栈槽内容。
4. 回调把两个 32 位部分重新组合成 64 位地址，得到指向原生栈附近的指针。

核心结构如下：

```javascript
// nop: () -> (i64, i64, i64)，函数体为空
// foo: (i64, i64, i64) -> ()，把前两个参数传给 JS
leak = [call nop, call foo]
```

这一步提供了 V8 沙箱外的 raw pointer。再配合上一节的 `read64`，可遍历原生栈并寻找 Wasm 已编译代码的地址。

### 4. 定位并改写 Wasm RWX 代码

官方 exploit 先调用一个包含多条常量赋值的 `main`，确保生成稳定的 Wasm 机器码；随后从泄露栈地址附近读取指针：

```javascript
let rwx = read64(stack_addr + 0x20n - 7n) - 0x40n;
let funcOff = (read64(rwx - 7n) >> 8n) & 0xffffn;
let mainCode = rwx + funcOff + 5n;
```

这些偏移只适用于随题提供的 V8 构建。定位 `main` 的代码后，用 `write64` 写入 `execve("/bin//sh", ...)` shellcode，再调用 `main()`，执行流便进入新代码。获得 shell 后运行 `/read_the_flag`；仓库环境中的 flag 为：

```text
RCTF{WebAssembly_is_fun}
```

服务层 exploit 还要按提示完成 PoW，发送 `exp.js` 的字节长度与正文。由于脚本限制为 100 KiB，而 V8 自带的完整 `wasm-module-builder.js` 加利用代码约 84 KiB，可以直接一并上传，无需依赖被删除的 `load` 接口。

### 5. 总 PDF 中的 Nu1L 变体

总 PDF 第 99 至 154 页完整粘贴了 `wasm-module-builder.js`；这些页面是生成 Wasm 二进制的通用辅助代码，并非 56 页不同的漏洞步骤。真正的利用代码位于第 155 至 159 页，路线与上述原理一致，但原语组织略有不同：

- `RandomLeak` 声明返回两个 `i64` 和两个 `f64`，函数体为空；预热 1000 次后再次调用，从虚假 `f64` 返回值取得残留栈指针。
- `addrOf` 声明 `(externref) -> i64`，却直接返回 `externref` 参数；`fakeObj` 反向声明 `(i64) -> externref`。两者都利用函数末尾 merge 类型检查被移除。
- 另建 `i64 -> ref struct(i64)` 的 caster，通过 `struct.get/set` 形成 `read64`、`write64`。
- 创建一个 victim Wasm 函数以获得已编译代码页，扫描泄露的栈区，从固定候选槽找到 RWX 指针。
- 先在代码页铺 NOP 和短跳转，再写入同一段 `/bin//sh` shellcode；随后执行 victim 函数进入载荷。

PDF 版本打印 1024 个栈槽用于调试，且把 RWX 候选固定为第 11 个槽；整理后的 exploit 应在附件版本上确认特征与偏移，不必保留这类大段诊断输出。

## 方法总结

- 根因是 Wasm merge point 的“栈元素数量检查”和“逐项类型检查”同时被移除；element section 与本题漏洞无关。
- arity 大于 1 会稳定进入被掏空的 slow path，可用空函数制造虚假多返回值；单返回值类型不匹配则可制造 `i64`、`externref`、WasmGC struct 引用之间的混淆。
- 完整利用链为：虚假返回值泄露原生栈地址，`i64 -> structref` 形成任意地址读写，栈扫描定位 Wasm 机器码，改写代码页并再次调用函数。
- V8 提交、构建选项、对象标记、Wasm 代码布局和栈槽偏移都由附件锁定。迁移到其它版本时应重新标定地址修正与代码页特征，不能直接复用数值偏移。
