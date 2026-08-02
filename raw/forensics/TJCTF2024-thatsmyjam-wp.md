# thatsmyjam

## 题目简述

题目给出 gzip 压缩的 Linux 磁盘镜像 `disk.img.gz`，提示内容与音乐有关。flag 不在普通文本配置中，而是用户 `avatar-kyoshi` 主目录下的一段可疑音频 `/home/avatar-kyoshi/music/sus.wav`。需要先做文件系统取证，再把音频中的摩尔斯电码转成文本。

## 解题过程

先解压镜像并只读挂载。若镜像包含分区表，应先用 `mmls` 或 `fdisk -l` 查出分区起始扇区，再乘以 512 得到挂载偏移；不要直接可写挂载取证源。

```bash
gzip -dk disk.img.gz
mmls disk.img
sudo mount -o ro,loop,offset=$((START_SECTOR * 512)) disk.img /mnt/tjctf
find /mnt/tjctf/home -type f | sort
cp /mnt/tjctf/home/avatar-kyoshi/music/sus.wav ./sus.wav
sudo umount /mnt/tjctf
```

播放或查看 `sus.wav` 的波形/频谱，可以听出长短两种音长：短音记为点，长音记为划；较短静音分隔同一字母内的符号，较长静音分隔字母和单词。按国际摩尔斯表解码后得到：

```text
hell0isitm3yourel00kingf0r
```

题面要求转换为小写并补上花括号，因此最终 flag 是：

```text
tjctf{hell0isitm3yourel00kingf0r}
```

## 方法总结

- 磁盘镜像题先确认分区偏移并只读挂载，保证证据不被主机自动写入污染。
- 可疑音频的文件路径本身就是重要线索；完成文件系统枚举后，应继续分析载荷语义，而不是把导出文件当作终点。
- 摩尔斯解码要同时利用音长和静音间隔。题面给出的大小写与包裹格式也必须执行，不能直接提交音频转写原文。
