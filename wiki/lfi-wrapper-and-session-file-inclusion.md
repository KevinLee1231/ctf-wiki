---
type: technique
tags: [web, lfi, php-wrapper, session, file-inclusion]
skills: [ctf-web]
raw:
  - ../raw/web/php-lfi-ssti-ssrf-and-type-juggling.md
updated: 2026-07-27
---

# LFI Wrapper and Session-File Inclusion

## 适用场景

服务端把可控路径传给 include/read/render 接口，可通过路径穿越、PHP wrapper、日志/session 临时文件或上传落点读取源码并在特定条件下推进到代码执行。

## 识别信号

- 参数控制模板、语言、页面或文件名，错误暴露真实路径。
- `php://filter`、`data://`、`phar://` 或归档 wrapper 未被统一禁用。
- Session、日志、上传临时文件包含攻击者可控内容。

## 最小证据

- 用固定系统文件或源文件证明读取/包含边界。
- 区分 read-only 文件读取与会执行脚本的 include。
- 定位可控文件的实际路径、生命周期和权限。

## 解法骨架

1. 确认路径拼接、后缀追加、规范化和 include API。
2. 先用 wrapper/编码读取源码，恢复目录与配置。
3. 若需要执行，寻找 session、日志、上传临时文件或可写缓存。
4. 写入最小标记并 include 验证，再控制命令执行副作用。

## 关键变体

- Wrapper source disclosure：filter 链读取 PHP 源码。
- Session/log poisoning：可控内容写入已知文件后被 include。
- Archive/PHAR：解析器或反序列化副作用参与执行。

## 常见陷阱

- 文件可读不等于会执行。
- 只猜默认 session 路径，未从配置或错误信息确认。
- 后缀追加和 NUL/URL 解码层次判断错误。

## 关联技巧

- [php-lfi-ssti-ssrf-and-type-juggling.md](php-lfi-ssti-ssrf-and-type-juggling.md)
- [url-parser-wrapper-and-ssrf-filter-differential.md](url-parser-wrapper-and-ssrf-filter-differential.md)
- [server-side-expression-and-command-context-escape.md](server-side-expression-and-command-context-escape.md)

## 原始资料

- [php-lfi-ssti-ssrf-and-type-juggling.md](../raw/web/php-lfi-ssti-ssrf-and-type-juggling.md)
