# DownUnderCTF 2020 - Twitter

## 题目简述

题目要求检查 DownUnderCTF 的公开 Twitter 账号并找到含 flag 的帖子。决定性证据来自比赛组织方的公开社交媒体，而不是本地附件；目标帖子中的内容仍需做一次 Base64 解码。

## 解题过程

从账号主页按时间追溯到其首条比赛相关帖子。官方 writeup 记录的原始证据页是 [DownUnderCTF status 1287018872457977856](https://twitter.com/DownUnderCTF/status/1287018872457977856)，其中出现：

```text
RFVDVEZ7aHR0cHM6Ly93d3cueW91dHViZS5jb20vd2F0Y2g/dj1YZlI5aVk1eTk0c30=
```

字符集和末尾 `=` 表明这是标准 Base64：

```bash
printf '%s' 'RFVDVEZ7aHR0cHM6Ly93d3cueW91dHViZS5jb20vd2F0Y2g/dj1YZlI5aVk1eTk0c30=' | base64 -d
```

输出即为完整 flag；其中的 URL 是 flag 字面内容，不能删改：

```text
DUCTF{https://www.youtube.com/watch?v=XfR9iY5y94s}
```

帖子中的有效信息已经完整转写到正文，即使社交平台页面以后不可访问，仍能复现解码步骤；因此不需要额外保存纯文本截图。

## 方法总结

- 核心技巧：从官方公开社交账号定位指定历史帖子，再识别并解码其中的 Base64。
- 识别信号：题面点名社交平台、账号或“first post”时，应记录精确账号、status ID 和原始线索文本，而不是只保存搜索结果页。
- 复用要点：OSINT 来源可能被删除或改版，WP 应同时保留直接证据 URL 与关键内容摘要；编码解码属于后处理，不能替代对来源真实性的核对。
