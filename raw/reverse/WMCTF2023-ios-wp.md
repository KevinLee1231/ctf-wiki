# ios

## 题目简述

题目是 iOS IPA 逆向。应用使用个人开发者证书签名，安装到越狱设备后会先触发越狱检测；绕过检测后，主校验逻辑位于 `ViewController` 的按钮点击处理函数。程序对输入执行类似 RC4 的变换，并与内置密文比对，最终恢复 flag。

## 解题过程

由于 IPA 使用的是个人开发者证书签名，需在越狱手机上安装后再配合 Cydia 插件（AppSync）跳过签名校验，流程较基础，此处不再展开。
IPA 安装成功后界面如下：

![IPA 安装后主界面显示输入框和 Verify 按钮](WMCTF2023-ios-wp/app-main-screen.png)

此时输入任意字母并点击 `verify` 时，会提示 `jailbreak device can not continue`，因此需要先绕过越狱检测。

![未绕过越狱检测时弹出 Jailbreak Detected 提示](WMCTF2023-ios-wp/jailbreak-detected-alert.png)

题目中明确有越狱检测。可直接安装一个屏蔽检测的插件来过掉，这里使用 **Liberty Lite**。在目标应用启用该插件后，可再次正常触发输入校验。进而进入二进制层：先解包 IPA，拿到 Mach-O 文件，再用 `class-dump` 导出该 macho 的全部 header。

![class-dump 导出的 ViewController 头文件定位按钮处理逻辑](WMCTF2023-ios-wp/classdump-viewcontroller.png)

拿到对应的 `ViewController` 后，可先粗看上层逻辑，`UIButton` 对应校验代码。使用 IDA 打开对应 Mach-O，同时用 Frida 进行 hook。
现在 hook 该 UI 的弹窗逻辑：

![Frida 准备 hook UIAlertView 初始化函数定位弹窗来源](WMCTF2023-ios-wp/frida-alert-hook-target.png)

监听 `[UIAlertView initWithTitle:message:delegate:cancelButtonTitle:otherButtonTitles:]`，并打印调用栈：

![UIAlertView hook 打印的调用栈指向 handleButtonClick](WMCTF2023-ios-wp/frida-alert-callstack.png)

可见上层调用来自 `-[ViewController handleButtonClick:]`，其地址是 `0x104e6bb00`，主模块基址为 `0x104e64000`，所以在 IDA 中对应偏移为  
`0x104e6bb00 - 0x104e64000 + 0x100000000 = 0x100007B00`。

![IDA 中根据偏移定位到 handleButtonClick 伪代码](WMCTF2023-ios-wp/ida-handle-button-click.png)

对应地址的伪代码如上，字符串显然是加密后的；继续向下追踪 `v48` 的调用，看到其调用关系如下：

![成功弹窗调用链指向输入校验后的分支](WMCTF2023-ios-wp/ida-success-alert-call.png)

推测该窗口即为 flag 正确时弹出的界面。由于题目中涉及 OLLVM，需逐步向上追溯可见：

![OLLVM 混淆中回溯到 sub_1000091e4 输入处理函数](WMCTF2023-ios-wp/ollvm-input-handler.png)

输入处理由 `sub_1000091e4` 完成，先用 hook 做确认。

![Frida hook 确认 sub_1000091e4 会处理用户输入](WMCTF2023-ios-wp/frida-confirm-input-handler.png)

确认无误后继续跟进该函数，在 IDA 中可见一段参数处理逻辑，`v59` 即输入参数：

![IDA 中确认 v59 是用户输入参数](WMCTF2023-ios-wp/ida-input-param-v59.png)

Frida hook 得到 `v120` 固定为 `"tfvq29bcom.runig"`，`v59` 的值是我们输入的 `qwe`：

![Frida 输出固定字符串 tfvq29bcom.runig 与测试输入 qwe](WMCTF2023-ios-wp/frida-rc4-input-key.png)

继续跟踪其他参数可见 `v104` 实际是输出地址：

![IDA 中确认 v104 是加密输出缓冲区](WMCTF2023-ios-wp/ida-output-buffer-v104.png)

另外可见输出长度和输入长度相同（`sub100004A9C` 的静态分析可确认），得到几个关键点：

1. 生成 `v6` 数组时涉及到 key；
2. `v6` 数组长度应为 255；

![IDA 中生成 RC4 状态数组时使用的 key 相关逻辑](WMCTF2023-ios-wp/ida-rc4-key-schedule.png)

![RC4 状态数组长度和初始化流程符合 255 字节特征](WMCTF2023-ios-wp/ida-rc4-state-array.png)

由此可推测是 RC4 算法，CyberChef 验证结果如下。

![CyberChef 验证 RC4 猜测可还原中间结果](WMCTF2023-ios-wp/cyberchef-rc4-check.png)

输出值应与预置数据比对。基于该特征去全局搜索 `v104` 的引用：

![全局交叉引用中定位 v104 输出缓冲区的比较位置](WMCTF2023-ios-wp/xref-output-buffer-v104.png)

对应的汇编地址是 `0x9D84`。

![汇编位置 0x9D84 对应输出与预置数据比较](WMCTF2023-ios-wp/asm-output-compare-site.png)

直接 inline hook 验证，确认 `x9` 确实是输出。

![inline hook 验证 x9 寄存器保存加密输出](WMCTF2023-ios-wp/inline-hook-output-register.png)

接下来要确认用于对比的数据，以及对应的 `x8` 寄存器是否来自内存，直接做 inline hook：

![inline hook 抓取 x8 指向的预置比较数据](WMCTF2023-ios-wp/inline-hook-expected-buffer.png)

再次验证 RC4 结果：

![CyberChef 再次验证 RC4 输出和预置数据关系](WMCTF2023-ios-wp/cyberchef-final-rc4-check.png)

数据是否乱码？目前数据长度是多少？

![内存窗口确认预置比较数据长度和字节内容](WMCTF2023-ios-wp/expected-buffer-length.png)

最终得到：

`wmctf{K3p1n2un#1n9!\$#@}` 

核验结果如下：

![输入最终 flag 后应用弹出 Congratulations 验证成功](WMCTF2023-ios-wp/ios-success-flag-alert.png)

## 方法总结

- 核心技巧：在越狱设备上结合 Frida hook 和 IDA 静态分析，定位 UI 校验函数并还原 RC4 类算法。
- 识别信号：iOS 题出现越狱检测、按钮校验、加密字符串和 OLLVM 混淆时，应先绕过环境检测，再 hook 弹窗/校验入口定位真实函数。
- 复用要点：移动端逆向不要只看伪代码；用 Frida 观察寄存器、参数和输出缓冲区，可以快速确认静态分析中的算法猜测。
