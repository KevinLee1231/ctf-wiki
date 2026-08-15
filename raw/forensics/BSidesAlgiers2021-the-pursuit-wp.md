# The Pursuit

## 题目简述

附件是一份约 17 MB 的网络抓包，需要从网络活动中找出被传输的证据文件，再处理其中的加密归档和嵌套载荷。实际链路为 SMB2 文件传输、ZIP 口令恢复、文件 carving。

## 解题过程

对抓包做协议层级统计，可以看到 12260 个数据帧中存在明显的 SMB2 会话。使用 Wireshark 过滤 smb2 后，可以在文件操作中定位到 Public/Downloads/CrackMe.zip；也可以直接通过“File → Export Objects → SMB”导出该对象。

ZIP 受口令保护。先转成 John the Ripper 可识别的哈希，再使用题目年代常见的 rockyou 字典：

~~~bash
zip2john CrackMe.zip > CrackMe.zip.hash
john --wordlist=/usr/share/wordlists/rockyou.txt CrackMe.zip.hash
~~~

恢复结果为：

~~~text
PINKPANTHER
~~~

解压后得到名为 whoami 的约 16 MB 文件。file 只能把它识别为普通 data，但 binwalk 在十进制偏移 10485760，也就是十六进制 0xA00000，识别出一个 640×197 的 GIF：

~~~bash
7z x CrackMe.zip -pPINKPANTHER
binwalk whoami
dd if=whoami of=flag.gif bs=1 skip=10485760 status=none
file flag.gif
~~~

打开 carving 出来的 GIF 即可读出：

~~~text
shellmates{Y0U_w0n_th3_pur5u1t}
~~~

本地对原 PCAP 做只读统计也确认了 SMB2 流量分支。官方解法中的 Wireshark 菜单、终端输出和最终文字 GIF 都可准确转成上述文本，因此没有复制这些截图。

## 方法总结

PCAP 题应先用协议统计缩小范围，再使用协议感知的对象导出，而不是直接对整包无目标地 strings。恢复归档后，扩展名和 file 结果仍可能只是外层伪装；继续用 binwalk 检查偏移，并记录准确 carving 参数，才能让证据链从网络会话一直闭合到最终载荷。
