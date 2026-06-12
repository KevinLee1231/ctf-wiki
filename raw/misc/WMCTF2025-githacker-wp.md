# GitHacker

## 题目简述

题目给出一个 Git 仓库，表面上 `git log` 只能看到不完整提交历史，但仓库曾经做过回滚、改密码、重新上传加密文件等操作。目标是从 Git 的引用日志中恢复历史提交，拿到挂载密码和两个加密容器 `image.png`、`image.jpg`，再利用 VeraCrypt 容器修改挂载密码时不会重加密数据区这一点恢复第二段 flag。

本题的关键识别信号有两个：一是 `git log` 不完整但 `.git` 目录还在，说明要看 `git reflog`；二是两个“图片”文件内容高熵、大小规整、普通图片解析失败，更像 VeraCrypt 这类加密卷容器，而不是 PNG/JPG 图片。

## 解题过程

直接使用 `git log` 拉不全日志，因为仓库做过回滚操作：

![这里直接使用 git log 命令是拉不全log日志的，做了部分回滚的操作](<WMCTF2025-githacker-wp/这里直接使用-git-log-命令是拉不全log日志的-做了部分回滚的操作.png>)

改用 `git reflog` 可以看到 HEAD 曾经指向过的提交，包含普通日志里被回滚隐藏掉的历史：

![git reflog 可以看到完整的日志](<WMCTF2025-githacker-wp/git-reflog-可以看到完整的日志.png>)

附件目录本身就是一个带隐藏 `.git` 的工作树，当前工作区只剩 `README.md`，并且 `image.png` 处于删除状态。reflog 中能直接看到关键提交：

```text
6ec92bc encryptedFile    # 首次加入 image.png
a026274 password         # 加入 password.md
93ab7b9 change password
d504bbf encryptedFile    # 加入 image.jpg
484c0d3 have a good time # 当前 HEAD
```

其中 `git show a026274:password.md` 可以取出旧挂载密码：

```text
EasyP@ssw0rd_from_Git_History
```

从 reflog 的行为顺序可以还原出主要操作：

```text
1. 上传一个加密文件；
2. 提交挂载密码；
3. 修改加密卷挂载密码；
4. 重新上传加密文件；
5. 通过回滚隐藏部分历史。
```

因此需要从相关历史提交中取回 `password` 和两个加密文件 `image.jpg`、`image.png`。做法可以是 `git checkout <commit> -- <file>`，也可以切到对应提交后复制文件；重点是不要只看当前工作区。

![先自由发挥把**password**和两个**加密文件**image.jpg、image.png拿到，我这里是用 git checkout](<WMCTF2025-githacker-wp/先自由发挥把-password-和两个-加密文件-image-jpg-image-png拿到-我这里是用-git.png>)

分析 `image.png`，文件内容看起来是高熵随机数据：

![分析一下**image.png**，乱七八糟的数据](<WMCTF2025-githacker-wp/分析一下-image-png-乱七八糟的数据.png>)

再结合十分规整的文件大小，可以判断它不是正常图片，而是加密容器：

![加上十分规整的文件大小](<WMCTF2025-githacker-wp/加上十分规整的文件大小.png>)

这里的 `vc` 指 VeraCrypt 容器。VeraCrypt 容器通常表现为无明显文件头、高熵、固定大小、无法被图片工具正常解析；题目把扩展名改成 `.png` / `.jpg` 是为了误导。

> 很多选手反应这个加密方式不清楚，个人出题时感觉这偏常识性知识点就并没有在题目里多给线索
>
> 现在想想也许把`commit`换成**encryptedContainer**会更好，但也稍微过于明显

用历史提交中恢复出的密码 `EasyP@ssw0rd_from_Git_History` 挂载 `image.png`，可以得到 `flag1`：

![直接使用之前获取的密码**EasyP@ssw0rd_from_Git_History**挂载加密容器**image.png**，得到 flag1](<WMCTF2025-githacker-wp/直接使用之前获取的密码-easyp-ssw0rd-from-git-history-挂载加密容器-image-p.png>)

![直接使用之前获取的密码**EasyP@ssw0rd_from_Git_History**挂载加密容器**image.png**，得到 flag1](<WMCTF2025-githacker-wp/直接使用之前获取的密码-easyp-ssw0rd-from-git-history-挂载加密容器-image-p-07.png>)

第一段内容为：

```text
WMCTF{G00d_J0b_F1nding_Th3_0ld_V3rsi0n_
```

结合 reflog 中“修改密码、重新上传文件”的顺序，可以推断 `flag2` 在 `image.jpg` 里。但直接用旧密码挂载 `image.jpg` 会失败，因为它的挂载密码已经被改过。

VeraCrypt 修改“挂载密码”时，通常不会重新加密整个数据区。它会保留数据区使用的主密钥 `Masterkey`，只重新生成/加密卷头中用于保护主密钥的那部分材料。这样做可以让大体积容器快速改密码，不必把几 GB 的数据全部重加密。

![但通过日志描述，加上一点实际操作可以猜测，出题人在修改密码的时候是直接修改了vc加密卷密码（事实上现实里大部分也都是这么操作的](<WMCTF2025-githacker-wp/但通过日志描述-加上一点实际操作可以猜测-出题人在修改密码的时候是直接修改了vc加密卷密码-事实上现实里大部分也.png>)

对比两个容器也能印证这个判断：除了卷头附近有变化，后面绝大多数数据区内容没有变化。这说明 `image.jpg` 很可能是在 `image.png` 的基础上只改了挂载密码，数据区主密钥仍然相同。

![通过对比两个容器的文件内容也可以印证这一点。除了容器卷头的部分数据被改变了，后面绝大多数的内容并没有变化](<WMCTF2025-githacker-wp/通过对比两个容器的文件内容也可以印证这一点-除了容器卷头的部分数据被改变了-后面绝大多数的内容并没有变化.png>)

VeraCrypt 容器可以粗略理解成两层密钥：

- `Masterkey`：创建容器时由随机熵生成，用于加解密真实数据区。
- 挂载密码：用户输入的密码，用来解密卷头，从卷头里取出 `Masterkey`。

因此可以先从能用旧密码打开的 `image.png` 备份卷头，再把该卷头恢复到 `image.jpg`。恢复后，`image.jpg` 的卷头又变成了“用旧密码保护同一个 Masterkey”的状态，于是可以用 `EasyP@ssw0rd_from_Git_History` 挂载 `image.jpg` 并读取第二段 flag。实际操作前应先复制一份 `image.jpg` 再恢复卷头，避免误操作破坏原文件。

![基于以上，由于 image.jpg 仅仅是在 image.png 的基础上修改了**挂载密码**，并没有修改 masterkey ，因此直接备份 image.png 的**加密卷头](<WMCTF2025-githacker-wp/基于以上-由于-image-jpg-仅仅是在-image-png-的基础上修改了-挂载密码-并没有修改-mast.png>)

![基于以上，由于 image.jpg 仅仅是在 image.png 的基础上修改了**挂载密码**，并没有修改 masterkey ，因此直接备份 image.png 的**加密卷头](<WMCTF2025-githacker-wp/基于以上-由于-image-jpg-仅仅是在-image-png-的基础上修改了-挂载密码-并没有修改-mast-11.png>)

![基于以上，由于 image.jpg 仅仅是在 image.png 的基础上修改了**挂载密码**，并没有修改 masterkey ，因此直接备份 image.png 的**加密卷头](<WMCTF2025-githacker-wp/基于以上-由于-image-jpg-仅仅是在-image-png-的基础上修改了-挂载密码-并没有修改-mast-12.png>)

![基于以上，由于 image.jpg 仅仅是在 image.png 的基础上修改了**挂载密码**，并没有修改 masterkey ，因此直接备份 image.png 的**加密卷头](<WMCTF2025-githacker-wp/基于以上-由于-image-jpg-仅仅是在-image-png-的基础上修改了-挂载密码-并没有修改-mast-13.png>)

第二段内容为：

```text
And_Y0u_M4ster_The_VeraCrypt_H34der_Trick!}
```

最终拼接得到：

```text
WMCTF{G00d_J0b_F1nding_Th3_0ld_V3rsi0n_And_Y0u_M4ster_The_VeraCrypt_H34der_Trick!}
```

## 方法总结

- 核心技巧：用 `git reflog` 找回被回滚隐藏的提交，再利用 VeraCrypt 卷头和数据区密钥分离的设计恢复改密后的容器。
- 识别信号：`git log` 历史明显缺失、提交信息出现密码/上传/回滚；疑似图片文件高熵且大小规整，无法按图片解析。
- 复用要点：Git 取证不要只看当前分支日志，reflog、stash、dangling object 都可能保留证据。VeraCrypt 改挂载密码不等价于重加密数据区，只要两个容器共享同一数据区主密钥，就可以通过恢复旧卷头让旧密码重新可用。
