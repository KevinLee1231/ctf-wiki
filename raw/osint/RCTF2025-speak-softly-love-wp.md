# Speak Softly Love

## 题目简述

附件是一段被降低画质并剪掉片尾的老式计算机音乐视频，需要通过画面中的播放器标识定位原始上传，再沿开发者 Mateusz Viste 的公开足迹回答 SVN 修订号、姓名发音文件 URL 和比特币捐赠地址。四问全部正确后得到最终 flag。

题目刻意破坏了常规反向识图条件，可靠路线是提取视频内的文字线索，再围绕“软件 → 原始视频 → 项目主页与版本库 → 开发者主页与 GopherSpace”建立同一身份链。

## 解题过程

### 1. Video ID：从画面识别 DOSMid

在附件视频约 `01:23` 处，右上角会短暂出现：

```text
DOSMID v0.8 loading
```

以 `DOSMid YouTube` 搜索，可以定位标题为 “DOSMid: The Godfather theme played on an 8086 computer” 的[原始上传](https://www.youtube.com/watch?v=8ssDGBTssUI)。它说明视频中的程序是 DOSMid，音乐是《The Godfather》的主题旋律 “Speak Softly, Love”，并给出题目要求的 YouTube 视频 ID：

```text
8ssDGBTssUI
```

### 2. Code Revision：追踪软错误循环修复

视频简介和公开搜索都可继续指向 [DOSMid 项目页](https://mateusz.fr/dosmid/)，作者是 Mateusz Viste，源码历史位于公开 SVN。题目问的是“为避免播放列表中不存在文件导致连续 soft error loop 而加入保护”的修订，直接查看日志：

```text
svn log svn://svn.mateusz.fr/dosmid
```

对应记录为：

```text
r178 | mv_fox | 2016-05-09 01:21:38 +0800 | 1 line

if too many 'soft' errors occur in a row, dosmid aborts
(protects against 'soft errors loops', typically with playlist
filled with non-existing files)
```

因此第二问按 SVN 格式提交 `r178`，不是只填数字 `178`。

### 3. Name-pronunciation URL：回到作者主页

Mateusz Viste 的[个人主页](https://mateusz.viste.fr/)在自我介绍区域直接提供姓名发音录音。题目要求的是完整 URL，因此第三问必须原样提交：

```text
https://mateusz.viste.fr/mateusz.ogg
```

该 `.ogg` 文件的作用是证明“Mateusz”这个名字如何发音，并把项目作者与个人站点绑定起来；它不是要继续做音频隐写。

### 4. Donation address：进入 GopherSpace

个人主页还给出作者维护的 GopherSpace。现代浏览器通常不再支持 Gopher 协议，可以用文本浏览器访问：

```text
lynx gopher://gopher.viste.fr/
```

在菜单中进入 `Donate som BTC bits`，可见公开的 Bitcoin 捐赠地址：

```text
16TofYbGd86C7S6JuAuhGkX4fbmC9QtzwT
```

这里应使用作者 GopherSpace 公布的地址，而不是把普通网页中的旧归档或其它项目地址当成答案。依次提交四问后得到：

```text
RCTF{wh3n_8086_s4ng_s0f7ly_0f_l0v3}
```

## 方法总结

- 压缩和裁剪会削弱反向识图，但不会消除视频内的 UI 文本；逐帧观察版本号、程序名和加载提示，比只截取主体画面更可靠。
- OSINT 多问应围绕同一实体逐层扩展来源：公开视频确认软件与作者，项目版本库回答变更历史，个人主页回答身份信息，GopherSpace 回答旧网络中的公开资料。
- 记录答案格式同样重要：YouTube 问的是 ID、SVN 问的是带 `r` 的 revision、发音问的是完整 URL、捐赠问的是原始地址。来源找对但截断格式仍会提交失败。
