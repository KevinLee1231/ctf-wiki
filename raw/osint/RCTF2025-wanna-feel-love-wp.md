# Wanna Feel Love

## 题目简述

这是一道按顺序解锁的五阶段综合题，附件为一封带 `challenge.xm` 的邮件。前两关要从伪装垃圾邮件和 XM sample 中提取隐藏文本，后三关则围绕 “I Feel Fantastic” 视频、歌唱机器人 Tara、开发者 John Bergeron 及其公开资料进行 OSINT。

平台源码表明五关答案必须依次提交；全部通过后才返回最终 flag。总 PDF 在第三关日期和第五关 URL 拼写上存在笔误，以下以仓库内实际校验器为准，并在相应位置说明差异。

## 解题过程

### 1. 垃圾邮件正文中的 SpamMimic 隐写

`.eml` 是 `multipart/mixed` 邮件，文本正文看似普通垃圾广告，附件则是 `challenge.xm`。第一关的“email trying to tell you”指向正文而非音频；把纯文本部分粘贴到 [SpamMimic Decode](https://www.spammimic.com/decode.shtml)，即可恢复：

```text
Don't just listen to the sound; this file is hiding an 'old relic.' Try looking for the 'comments' that the player isn't supposed to see.
```

这段明文既是第一关答案，也明确提示下一关要检查音频容器中“播放器看不到的 comments/旧遗物”。链接只承担解码器入口作用；其功能和解码结果已在此完整记录。

### 2. XM 第五个 sample 的波形比特

用 OpenMPT 检查 XM 结构，可见标题 `How Do you Feel?`、tracker 名称 `the OpenMPT knows`，以及提示：

```text
They say if you trace the peaks carefully enough, it spells a sentence that was never meant to be heard.
```

五个 sample 中，第五个 `Feel` 从未被曲目调用，波形却呈现规则的高低能量块，这是实际隐写载体。将它无损导出后按以下步骤恢复：

1. 按原采样率每 `0.05 s` 切一段，计算每段 RMS 能量；
2. 对 RMS 排序，取相邻值最大间隙两端的中点作为动态阈值；
3. 高于阈值记为 `1`，否则记为 `0`；
4. 从头开始每 8 bit 按高位在前转成 ASCII。

结果为：

```text
I Feel Fantastic heyheyhey
```

不要对整个 XM 混音做能量切片；正常乐器会污染门限。必须先定位并导出未使用的第五个 sample，而且应避免 MP3 等有损转码改变高低能量边界。

### 3. 追溯原始 YouTube 上传

第二关明文对应著名的 Tara 歌唱机器人视频。已删除原视频的 [Ghostarchive 归档](https://ghostarchive.org/varchive/rLy-AwdCOmI)仍暴露视频资源名、上传者和日期元数据，可确定：

```text
Video ID: rLy-AwdCOmI
Uploader: Creepyblog
```

日期需要额外交叉验证。归档展示可能因时区或页面解析显示 `Thu, 16 Apr 2009`，但更早的页面记录及 [Mallory Marlowe 对该视频来源的整理](https://mallorymarlowe.substack.com/p/4-i-feel-fantastic-hey-hey-hey)都指向 Creepyblog 于 2009-04-15 上传。更关键的是，仓库内平台源码的实际校验值为：

```text
Upload Date: 2009-04-15
```

总 PDF 最后的答案列表把日期写成 `2009-04-16`，与它前文“该值提交不对”的叙述及平台校验器均冲突，应视为 WP 笔误，不能照抄。

### 4. 确认 DVD 购买入口、发件人和创作年份

[Android Music Videos 商品页](https://androidworld.com/prod68.htm)说明 John Bergeron 制作了一张约 15 分钟的歌唱机器人 DVD，包含五段各约 3 分钟的曲目，并在页面底部提供购买入口。题目要求提交的购买链接就是该商品页本身：

```text
https://androidworld.com/prod68.htm
```

页面留下的联系/PayPal 收款邮箱是 `crwillis@androidworld.com`，但题目字段问的是收到实体包裹时的 `Sender`，校验器也不接受邮箱。[Yitzi Litt 的实购调查](https://yitzilitt.medium.com/the-story-behind-i-feel-fantastic-tara-the-singing-android-and-john-bergeron-fc83de9e8f36)记录了几个月后收到的 DVD 和随包纸条，由此确认发件人姓名为：

```text
Chris Willis
```

同一调查还检查了 DVD 文件元数据，其中修改时间是 `December 6, 2004 at 9:29 PM`。这比笼统的“2003–2004 年间制作”更能支撑题目所需的单一年份：

```text
Creation Year: 2004
```

### 5. 定位开发者的数字墓碑

继续沿 John Bergeron 的全名、讣告和墓园线索检索，可定位 [Find a Grave 纪念页](https://www.findagrave.com/memorial/63520325/john-louis-bergeron)。题目要求 URL 不带末尾斜杠，平台源码中的精确答案为：

```text
https://www.findagrave.com/memorial/63520325/john-louis-bergeron
```

总 PDF 将 slug 写成 `john_louis-bergeron`，但实际页面和校验器使用 `john-louis-bergeron`，这里同样应以平台源码为准。完成五关后返回：

```text
RCTF{sh3_ju5t_f33ls_l0v3_thr0ugh_w1r3s_4nd_t1m3}
```

## 方法总结

- 多媒体题不要只播放成品：邮件正文、容器注释、未被调用的 sample、波形能量和文件元数据都可能是独立通道。先检查结构，再决定是否需要频谱或信号分析。
- 对二值波形用固定时窗统计 RMS，再从数据本身寻找两簇间阈值，比手写固定振幅门限更耐采样幅度变化；同时保留时窗、bit 顺序和字节端序，确保过程可复现。
- OSINT 多来源冲突时，要区分页面展示时间、二手文章、题目叙述与实际校验值。本题两个官方 WP 笔误均可由随题平台源码直接裁决，不能为了“忠实 PDF”保留错误答案。
