# miniup

## 题目简述

原始题解说明 PHP 图片预览功能把 `filename` 直接交给 `file_get_contents`，并将结果作为 Base64 图片返回；调用还把用户提供的 `options` 原样传给 `stream_context_create`：

```php
$file_content = @file_get_contents($filename, false, @stream_context_create($_POST['options']));
```

环境中的 dufs 文件服务可处理 HTTP PUT。于是 SSRF 不仅能读内部资源/源码，还能以可控 stream context 向本地 dufs 上传 PHP 文件，再通过其公开路径触发执行并读取环境变量中的 flag。

## 解题过程

### 从预览接口转为内网 PUT

先以本地地址作为 `filename` 确认 PHP 发起请求。随后指定 `options[http][method]=PUT`，把 PHP 文件作为 `options[http][content]`，目标定位为 dufs 的本地监听地址与可公开目录，例如：

```http
POST /index.php
Content-Type: application/x-www-form-urlencoded

action=view&filename=http://127.0.0.1:5000/i.php&options[http][method]=PUT&options[http][content]=%3C%3Fphp+echo+getenv%28%27FLAG%27%29%3B%3F%3E
```

`file_get_contents` 会按 stream context 发出 PUT，dufs 保存 `i.php`。之后浏览器请求该文件的可执行 PHP 路径，以 `getenv('FLAG')` 输出 flag。若 dufs 仅静态提供文件，必须先根据题目源码确认 PHP 解释器与 dufs 的目录/反向代理关系；不能仅凭上传成功就断言 RCE。

### 验证与证据边界

原始题解给出了上述关键调用和 payload，但当前源范围没有 miniup 的 `index.php`、dufs 配置、运行拓扑或响应记录。因此已知结论是“可控 stream context 产生 SSRF/内网 PUT”；PHP 文件是否由同一路径执行、以及 flag 回显细节在当前材料中未能独立复核，本文不虚构运行结果。

## 方法总结

- 核心技巧：将可控 `file_get_contents` stream context 从读型 SSRF 扩展为 HTTP PUT 内网写入。
- 识别信号：`file_get_contents`/`fopen` 接收 URL，且 `stream_context_create` 直接接受请求参数；内部服务存在上传或对象写入语义。
- 复用要点：网络访问应使用固定 URL/方法的专用客户端，禁止把请求参数直接作为 stream context；内网文件服务不应与 PHP 可执行目录共享。
