# Swift Surprise

## 题目简述

附件是一张能正常显示的 JPEG。图像内容本身只是载体，没有需要辨认的空间或视觉线索；flag 被作为可打印命令字符串追加在文件数据中，因此无需归档装饰性原图。

## 解题过程

先对文件提取可打印字符串：

~~~bash
strings -a Swift_Folklore.jpg | tail
~~~

在文件尾部可以看到：

~~~text
/bin/bash -c "wget http://maple{Use_5tr1ngs_2_f1nd_teXt}"
~~~

其中 URL 主机位置直接嵌入了 flag：

~~~text
maple{Use_5tr1ngs_2_f1nd_teXt}
~~~

用十六进制查看器检查 JPEG 的 EOI 标记 0xffd9 后也能确认这些字节是附加数据，而不是像素编码的一部分。

## 方法总结

媒体文件能正常打开并不代表结尾没有附加数据。快速检查顺序可以是 file、exiftool、strings、十六进制尾部与 binwalk。本题 strings 已能完整恢复信息，不应为了“有图片”而保留没有解题价值的截图或原始人物照片。
