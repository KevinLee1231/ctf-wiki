# Shadows of Asgard

## 题目简述

取证题围绕 PCAP、伪装落地页、加密 C2 JSON、PNG 隐写 chunk 和解密后的任务输出。逐项恢复公司名、注入程序路径、任务 ID、时间信息和最终 base64 flag。

## 解题过程

### 关键观察

取证题围绕 PCAP、伪装落地页、加密 C2 JSON、PNG 隐写 chunk 和解密后的任务输出。

### 求解步骤

Q1: 渊恒科技
Q2: C:\Users\dell\Desktop\Microsoft VS Code\Code.exe
Q3: c0c6125e
Q4: 2018-09-14 23:09:26
Q5: RCTF{they always say Raven is inauspicious}
Merchant’s Mask – Exported / from the pcap (http_objs/%2f) to read the fake landing page.
The title, headers, and copy all brand the site as 渊恒科技 (Yuanheng Technology), revealing
the company Loki used as camouflage.
Parasite’s Nest – Decrypted the C2 JSON blobs (e.g., 4ea5e4087fa4d2da*.tmp) with the
recovered AES key/IV. Multiple task outputs report the working directory as
C:\Users\dell\Desktop\Microsoft VS Code, and the directory listing shows Code.exe in place,
indicating Loki injected his agent into that VS Code installation.
Hidden Rune – Steganographic payloads in the logo_903830abfe618b5b*.png files (tEXt
chunks) decode to the attacker’s task metadata. The entry for the pwd command lists
taskId":"c0c6125e".
Forge of Time – A decrypted task response (4ea5e4087fa4d2da(4).tmp) records the drive
probe results, showing the C: drive was created on Friday, September 14, 2018 at 23:09:26
GMT‑0700 (PDT) and last modified Nov 12, 2025.
Raven’s Ominous Gift – Loki’s final upload task includes the base64 file contents. Decoding
UkNURnt0aGV5IGFsd2F5cyBzYXkgUmF2ZW4gaXMgaW5hdXNwaWNpb3VzfQ== yields the
hidden message RCTF{they always say Raven is inauspicious}.

## 方法总结

- 核心技巧：PCAP 对象导出、AES C2 解密、PNG chunk 隐写。
- 识别信号：流量中出现伪装网页、`.tmp` 加密任务结果和可疑图片元数据。
- 复用要点：把每个问题映射到具体 artifact，保留能证明答案的证据链。
