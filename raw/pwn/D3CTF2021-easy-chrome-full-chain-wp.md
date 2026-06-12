# EasyChromeFullChain

## 题目简述

本题是 RealWorld 浏览器全链题，目标是先在 Chrome 渲染进程中利用 V8 Turbofan 优化漏洞获得沙箱内 RCE，再利用自定义 Mojo 接口对象生命周期问题完成沙箱逃逸，最终在沙箱外执行指定程序读取 flag。WP 正文保留了 V8 类型推导错误、Mojo 生命周期 UAF 和关闭沙箱后的二次 RCE 条件；外部链接只作为题目来源和延伸参考。

## 解题过程

相关链接包括微信文章 <https://mp.weixin.qq.com/s/Gfo3GAoSyK50jFqOKCHKVA>、可访问镜像 <https://cn-sec.com/archives/1988035.html> 和题目附件镜像 <https://mega.nz/file/1zAERJ5R#M0R_7PAJhIailzqrYKTY-CR36F8J2vX6AYbVML0AHy4>。题目由 V8 Turbofan `MinusZero` 类型处理漏洞和自定义 Mojo `AntNest` 生命周期 UAF 两段组成，下面直接展开两段漏洞原理和衔接条件。

第一段是 V8 RCE。漏洞改编自 Chromium bug tracker 中的 Turbofan 优化问题，关键点在 `Simplified Lowering` 阶段判断 `SpeculativeSafeIntegerAdd/Sub` 是否可能溢出时，`CanOverflowSigned32` 对包含 `MinusZero` 的类型处理不完整。`Type::Intersect(..., Type::Signed32(), ...)` 会把 `-0` 信息丢掉，使优化器误判 `-0 - (-0x80000000)` 不会溢出，从而把节点 lower 成没有溢出检查的 `Int32` 运算。

利用时先构造 JIT 函数，让类型传播认为某个索引范围只会落在合法区间，实际运行时因为 `-0` 相关溢出让索引扩大。题目为了降低难度去掉了 bounds-check elimination hardening，因此可以让数组访问的 `CheckBounds` 被消除，进而构造越界读写。常见落点是让 double array 和 tagged array 相邻，分别得到 `addrof` 和 `fake_obj` 原语，再伪造 `ArrayBuffer` 得到任意地址读写。后续泄露 WASM 实例的 JIT RWX 页地址，把 shellcode 写入该区域并调用导出函数，完成渲染进程内 RCE。

第二段是 Chrome 沙箱逃逸。题目增加了 `antctf.mojom.AntNest` 接口，`Store` 和 `Fetch` 本身逻辑不复杂，真正的问题在生命周期：`AntNestImpl` 由 `mojo::MakeSelfOwnedReceiver` 绑定到通信管道，但对象内部保存了 `RenderFrameHost` 原始指针。如果先在 sub frame 里绑定接口，再释放该 sub frame，之后继续通过已保存的接口调用 `Store` 或 `Fetch`，就会在 `render_frame_host_->GetFrameDepth()` 处触发 UAF。

为了能在 sub frame 中使用 MojoJS，利用 V8 任意读写定位并遍历 `g_frame_map`，把对应 `RenderFrameImpl` 的 `enabled_bindings_` 改成允许 Mojo WebUI 绑定，同时让 `IsMainFrame()` 判断通过。触发 UAF 后，受控的 `RenderFrameHostImpl` 虚表调用可以进一步转化为回调对象调用：通过泄露 `chrome.dll` 基址、定位 `command_line`，伪造 `base::internal::BindState`，最终调用 `sandbox::policy::SetCommandLineFlagsForSandboxType(command_line, 0)` 关闭沙箱。关闭沙箱只对新渲染进程生效，因此最后需要在不同源页面中再次走 V8 RCE 执行 shellcode。

## 方法总结

这题的核心不是单点漏洞，而是浏览器全链利用的衔接：先把 JIT 类型推导错误转成稳定的 V8 任意读写，再用该能力开启 MojoJS、整理 Browser 进程对象地址和虚函数调用面，最后把 Mojo 生命周期 UAF 转成关闭沙箱的函数调用。

复盘时应重点保留三个检查点：`MinusZero` 是否被类型系统错误丢弃、数组 bounds check 是否真的被消除、UAF 触发时能否稳定控制 `RenderFrameHostImpl` 相关对象。沙箱逃逸部分不要只记“Mojo UAF”，还要明确 raw pointer 生命周期、sub frame 释放顺序、`BindState` 伪造位置和关闭沙箱后必须换新进程执行 RCE 这几个条件。

