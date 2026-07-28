---
type: technique
tags: [web, upload, polyglot, content-type, parser-confusion]
skills: [ctf-web, ctf-stego]
raw:
  - ../raw/web/ruby-php-upload-and-ssti-rce.md
  - ../raw/web/polyglot-url-tricks-and-ssrf-leaks.md
  - ../raw/web/0xGame2023-week2-ez-upload-wp.md
updated: 2026-07-28
---

# Upload Polyglot and Content-Type Confusion

## 适用场景

上传链的扩展名、MIME、magic、解码器、存储路径和后续解释器采用不同判定，攻击者可构造同时满足检查器和危险解析器的文件。

## 识别信号

- 客户端/代理/后端分别检查文件名、`Content-Type`、magic 或图像解码。
- 上传后文件被重命名、转码、解压、模板渲染或脚本 include。
- 同一字节可被图片/PDF/ZIP/脚本等多个解析器接受。

## 最小证据

- 建立上传、校验、存储、访问和二次处理的完整链路。
- 确认每层实际使用的判定依据和文件路径。
- Polyglot 必须分别通过两种目标解析器验证。

## 解法骨架

1. 先上传最小合法文件，记录响应、落盘名和访问方式。
2. 单变量测试扩展名、MIME、magic、multipart filename 和尾随数据。
3. 按严格解析器保留结构，把第二载荷放在宽松解析器可达位置。
4. 验证后续解释/解压/包含行为，而不止“上传成功”。

## 关键变体

- Extension/MIME mismatch。
- Image/document polyglot 与尾随 payload。
- Archive traversal、symlink 或解压后二次解释。

## 常见陷阱

- 上传成功但文件不可访问或不会执行。
- 只在本地 `file` 通过，目标解码器实际拒绝。
- 忽略服务端重编码会删除附加数据。

## 关联技巧

- [ruby-php-upload-and-ssti-rce.md](ruby-php-upload-and-ssti-rce.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](polyglot-url-tricks-and-ssrf-leaks.md)
- [file-format-and-embedded-payload-identification.md](file-format-and-embedded-payload-identification.md)

## 原始资料

- [ruby-php-upload-and-ssti-rce.md](../raw/web/ruby-php-upload-and-ssti-rce.md)
- [polyglot-url-tricks-and-ssrf-leaks.md](../raw/web/polyglot-url-tricks-and-ssrf-leaks.md)
- [0xGame2023-week2-ez-upload-wp](../raw/web/0xGame2023-week2-ez-upload-wp.md)
