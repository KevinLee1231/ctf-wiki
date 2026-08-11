# 玩玩条码

## 题目简述

附件包含 `JPNPostCode.png`、`7zipPasswordHere.mp4` 和加密的 `Code128.7z`。日本邮政四状态条码给出视频隐写密码；MSU StegoVideo 从 MP4 中提取 7z 密码；解压后再扫描 Code 128 得到 flag。

## 解题过程

日本邮政条码使用四种竖条状态。按“长条、上半长条、下半长条、短条”分别记为 1、2、3、4，每个数字对应三个状态：

```text
0 -> 144    1 -> 114    2 -> 132    3 -> 312    4 -> 123
5 -> 141    6 -> 321    7 -> 213    8 -> 231    9 -> 411
```

起始标志为 `13`，无字母时的填充符为 `423`。按三条一组读取附件中的有效数字区，得到：

```text
1087627
```

接着安装 32 位 VirtualDub2、FFmpeg Input Plugin 和 MSU StegoVideo，并确保插件位于 32 位插件目录。载入 `7zipPasswordHere.mp4` 后依次选择：

```text
Video -> Filters -> Add -> MSU StegoVideo 1.0
```

在插件中选择 `Extract file from video`，密码填写 `1087627`，指定输出文件。返回主界面，将进度条置于开头，再执行 `File -> Preview filtered`；播放前几秒即可完成提取。输出文本给出：

```text
Zip Password: b8FFQcXsupwOzCe@
```

用该密码解开 `Code128.7z`，得到 `Code128.png`。使用支持 Code 128 的离线扫码器读取，结果为：

```text
hgame{9h7epp1fIwIL3fOtsOAenDiPDzp7aH!7}
```

## 方法总结

- 核心链路：日本邮政四状态条码、MSU StegoVideo 文件提取、7z 解密、Code 128 解码。
- 识别信号：附件名已经把每层用途串起来；`JPNPostCode` 不是普通一维条码，它用条形高度而非宽度编码状态。
- 复用要点：VirtualDub 插件位数必须与主程序一致；视频隐写提取通常要实际运行预览滤镜，单纯添加滤镜不会生成文件。
