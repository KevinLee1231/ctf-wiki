# UMDCTF 2019 - LibTest

## 题目简述

题目提供一个 Linux 图形程序和字体文件。程序通过 SDL 创建窗口并绘制二维码，flag 不以可搜索的明文直接输出到终端。

## 解题过程

`file`、动态依赖和导入符号可以确认程序使用 SDL 图形库。直接运行时会弹出窗口；在无桌面环境中，也可以用虚拟 X 服务器捕获窗口：

```bash
Xvfb :99 -screen 0 1024x768x24 &
export DISPLAY=:99
./challenge &
sleep 1
import -window root screen.png
zbarimg --quiet screen.png
```

窗口中央显示的二维码如下：

![SDL 程序运行后绘制的 flag 二维码](./UMDCTF2019-libtest-wp/displayed-flag-qr.png)

二维码解码结果为：

```text
UMDCTF-{Y3s_7he_ch4ll_is_tHaT_eaSY}
```

其 SHA-256 与官方摘要一致。

## 方法总结

逆向题的输出渠道不一定是标准输出。看到 SDL、字体或窗口相关依赖时，应实际观察渲染结果；在服务器环境中可用 Xvfb 保留可复现的截图。二维码本身有视觉信息价值，因此保留语义化命名的关键截图，而不把它替换成没有上下文的纯文本。
