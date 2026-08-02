# linkedout-recon

## 题目简述

题目只给出一页简历 PDF。逐页视觉核对与文本层一致：姓名是 `Alex Marmaduke`，简历还写有 “Assisted on DEFCON-2023 logistics; contributed to Red Team documentation”，邮箱为 `thisisafakeemail@tjctf.org`。这些身份与 DEFCON 关键词是公开信息检索入口；PDF 元数据中的制作者姓名属于无关线索，不应转而调查真实出题人。

## 解题过程

搜索精确姓名后可定位到 [GitHub 账号 `ctf-researcher-alex`](https://github.com/ctf-researcher-alex/)。账号展示名同为 Alex Marmaduke，简介包含 DEFCON、SIGINT 与 OSINT，仓库 README 末尾还有 `DEFCON 2023 Notes`。其唯一的 [`dc2023-notes.md` gist](https://gist.github.com/ctf-researcher-alex/d258b99e29413793847f1d788967fede) 再次给出同一笔记入口，因此这不是普通同名用户。

笔记短链原先跳转到 Notion 的 `SIGINT Workflow Summary`；该 Notion 页面目前已经返回 404，但比赛期间的页面继续指向 Google Drive 上的 [`protected.zip`](https://drive.google.com/file/d/1LAh1UUpHlfeagrN72dL_M9AsS8PRRGVz/view)。归档内只有 `encoded.png`，7-Zip 信息显示加密方式为传统 `ZipCrypto`、压缩方式为 `Store`。

官方路线是对 ZIP 做常用口令字典攻击：

```bash
zip2john protected.zip > protected.hash
john --wordlist=/usr/share/wordlists/rockyou.txt protected.hash
unzip -P princess protected.zip
```

恢复出的口令是 `princess`。随后检查 PNG 的位平面：

```bash
zsteg encoded.png
```

决定性输出为：

```text
b1,rgb,lsb,xy .. text: "29:marmaduke:tjctf{linkedin_out}"
```

因此 flag 为：

```text
tjctf{linkedin_out}
```

即使 Notion 或云盘以后失效，以上正文仍保留了完整公开链路、文件名、加密方式、口令和图像通道，不需要重新打开外链才能理解解法。

## 方法总结

- 核心技巧：从文档中的合成人物身份 pivot 到匹配的 GitHub 资料，再沿 README、gist、Notion 和云盘逐级追踪公开 artifact。
- 识别信号：罕见全名、简历与 GitHub 简介共享 DEFCON/OSINT 语义，以及新建账号只留下一个高度相关 gist。
- 复用要点：OSINT 每个跳转都要用上一层的独立字段交叉验证；下载到加密包后及时转入格式识别和位平面提取，不要继续无目的搜索。
