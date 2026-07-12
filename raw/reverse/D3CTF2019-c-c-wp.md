# c+c

## 题目简述

题目提示 flag 在 readme 中，但十六进制查看只能在文件尾看到私用区字符 `U+E000`。该字符的显示由字体自行定义，因此真正的信息藏在游戏内加载的字体中。程序脱 UPX 后可见调用 `AddFontMemResourceEx` 动态加载字体，并在调用链上做 inline hook，hook 中解密内存并替换字体参数。解法是跟进 hook，dump 解密后的字体，安装/切换字体后读取私用区字符显示出的 flag。

## 解题过程

(这道题当初应该叫magic font的)

根据readme文件，flag就在此文件中。

用十六进制编辑器检查发现文件尾部有特殊字符.用UTF-8解码得到字符U+E000。

查阅Unicode定义发现此字符为 [私用区](https://zh.wikipedia.org/wiki/%E7%A7%81%E7%94%A8%E5%8D%80)，需要字体自行实现。

> \-><- 就是这个字符,可能根据浏览器以及字体设置的不同显示不同的内容

所以需要获取到字体。

显然这里指的是游戏内的字体。

脱掉upx壳,对程序逻辑就行逆向。

发现调用了gdi32中的AddFontMemResourceEx函数，下断点。

继续运行游戏，断下时发现有inline hook的特征，跟进。

(对程序处理Fonts.arc的逻辑就行逆向,或者直接在调用入口处dump都是陷阱)

(注：win10上的gdi32是gdi32full的wrapper，需要跟进第一个 `jmp` 才能发现hook)

在hook里面对一段内存就行了解密并替换了原函数参数。

dump出来再安装到系统里，在记事本中切换字体即可看到flag.

(看不清楚就放大放大再放大)

## 方法总结

- 核心技巧：私用区字符需要配套字体解释，看到不可见/异常 Unicode 字符时要找实际使用的字体资源。
- 识别信号：文件中出现 `U+E000` 等 PUA 字符、题目有游戏程序和字体加载逻辑时，应检查 `AddFontMemResourceEx`、字体包解密和 GDI hook。
- 复用要点：不要只 dump 静态 `Fonts.arc`，动态 hook 可能在调用入口处替换了解密后的字体内存；Win10 上 `gdi32` 可能只是 `gdi32full` wrapper，需要跟进跳转。

