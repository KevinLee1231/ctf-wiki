# GreyCTF2024 Out in Plain Sight WP

## 题目简述

题目指出 flag 在 Greyhats 宣传视频的少数帧中短暂出现，并给出正确 flag 的 MD5：`36ed337d208d4d58679cbb5047885236`。视频中还出现了多个假 flag，因此既要逐帧取证，也要用摘要筛选真值。

## 解题过程

从题目给出的 `@nus.greyhats` 账号定位当时的宣传 Reel。为了避免播放器跳帧，把视频保存后逐帧导出：

```bash
mkdir -p frames
ffmpeg -i reel.mp4 -vsync 0 frames/%06d.png
```

快速浏览缩略图并放大含 `grey{` 的帧，会看到不止一个候选。将每个候选按画面中的大小写原样抄录，然后计算 MD5；例如：

```bash
printf %s 'grey{y0uR_eYeS_aRe_5hArP}' | md5sum
```

输出为：

```text
36ed337d208d4d58679cbb5047885236
```

与题面摘要完全一致，因此正确 flag 是：

```text
grey{y0uR_eYeS_aRe_5hArP}
```

仓库没有保留原视频，正文已将公开解题记录中的关键事实——宣传 Reel、短暂帧、多个假 flag 和 MD5 复核——完整转写；纯文本 flag 帧不再作为低信息量截图归档。

## 方法总结

视频中一闪而过的文本应通过无丢帧导出检查，不能依赖网页播放器暂停。题面给出的摘要不是装饰，而是候选消歧的权威校验；计算时必须避免末尾换行，并保留 flag 的原始大小写。
