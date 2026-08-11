# Horoscopes

## 题目简述

题目只给出一个网络端口和一封暗示星座的邮件：新生儿是 Taurus，父母比它晚一个月，对应 Gemini。连接端口时返回的 `59 Invalid URL` 也符合 Gemini 协议的两位状态码格式。目标是识别并使用 Gemini capsule，而不是把服务当成 HTTP 或普通交互式 TCP 程序。

## 解题过程

Gemini 客户端向服务器发送一行 URL，响应首行为两位状态码和元信息；成功响应通常形如：

```text
20 text/gemini\r\n
```

可以用支持 Gemini 的客户端打开题目给出的主机与端口，也可以用 TLS 客户端手工发送请求：

```bash
printf 'gemini://<challenge-host>/\r\n' |
  openssl s_client -quiet -connect <challenge-host>:<port>
```

忽略赛事自签名证书只适用于这个 CTF 环境；真实场景仍应核验 capsule 证书。成功后，Gemtext 使用以 `=>` 开头的行表示链接，首页可以继续浏览各个 `.gmi` 页面。

当前官方源码、题目 hint 与 `challenge.yaml` 一致：flag 位于首页 `index.gmi` 的 “Message of the Day” 中，而不是必须依赖页面渲染或脚本执行：

```text
DUCTF{g3mini_pr0t0col_s4ved_us}
```

官方短解曾写成在 `salvaging.gmi` 底部找到，但仓库中的实际内容和提示均指向首页；本 WP 采用可核对的当前源码位置。

## 方法总结

- 核心技巧：从协议状态行和题面双关识别 Gemini，再按 Gemini URL 请求与 Gemtext 链接语法浏览 capsule。
- 识别信号：裸端口返回 `59 Invalid URL`，题面反复暗示 Gemini，且普通 HTTP 客户端无法正常解释响应。
- 复用要点：未知端口先观察原始字节和状态行；Gemini 成功码是 `20`，页面链接以 `=>` 开头。不要因为界面像网页就默认协议是 HTTP。
