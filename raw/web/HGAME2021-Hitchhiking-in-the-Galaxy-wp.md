# Hitchhiking_in_the_Galaxy

## 题目简述

题目通过连续的 HTTP 响应提示考察请求方法和请求头。访问 `HitchhikerGuide.php` 时会先遇到 `302` 跳转；直接请求目标页面又返回 `405 Method Not Allowed`。后续页面依次要求使用正确方法、伪装成“无限非概率引擎”、从 `Cardinal` 页面跳转而来，并让请求看起来源自本机。

## 解题过程

不要让客户端自动跟随 `302`，直接抓取 `HitchhikerGuide.php` 的原始响应。`405` 表明当前方法不被接受，逐一测试常见方法后可确认应改为 `POST`。

随后按照每次响应中的提示补齐请求头：

```http
POST /HitchhikerGuide.php HTTP/1.1
Host: challenge.example
User-Agent: Infinite Improbability Drive
Referer: Cardinal
X-Forwarded-For: 127.0.0.1
```

其中 `User-Agent` 对应无限非概率引擎，`Referer` 满足来源检查，`X-Forwarded-For` 则让应用层认为请求来自本机。若环境要求完整来源 URL，应以响应提示给出的 `Cardinal` 地址替换示例值。四项条件全部满足后得到：

```text
hgame{s3Cret_0f_HitCHhiking_in_the_GAl@xy_i5_dOnT_p@nic!}
```

官方 PDF 未抄录服务器返回值，上述结果通过 [Zry.IO 的同期 Week 1 复盘](https://zry.io/zh/cybersec/ctf/hgame2021-week-1-writeup/) 补齐；解题所需请求链已经完整写在正文中。

## 方法总结

这是一道按响应状态逐步修正 HTTP 请求的基础题。看到 `302` 应先关闭自动跳转并检查中间响应，看到 `405` 应检查请求方法；题面中的设备、来源和本地访问提示通常分别对应 `User-Agent`、`Referer` 与代理来源地址头。构造请求时应同时确认后端是否信任 `X-Forwarded-For`，真实系统中不能默认该头可代表客户端地址。
