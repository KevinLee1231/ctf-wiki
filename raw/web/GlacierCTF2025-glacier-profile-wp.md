# GlacierCTF 2025 Glacier Profile

## 题目简述

题目用 PHP 实现一个个人资料页，管理员入口比较 `/rcon.pw` 中随机生成的 32 字符密码，登录成功后显示 `/flag.txt`。镜像安装了 PHP-SPX profiler，并为其 Web UI 生成随机访问 key。

未授权的 `action=dbg` 会调用 `phpinfo()`，直接泄漏 `spx.http_key`。取得 profiler UI 后，可以读取每次登录请求的 `recorded_call_count`。密码比较逐字符调用独立的 `check_char` 并在首个不匹配处返回，因此函数调用数准确泄漏正确前缀长度。

## 解题过程

### 1. 从 phpinfo 取得 SPX key

向首页发送：

```http
POST /
Content-Type: application/x-www-form-urlencoded

action=dbg
```

`handleDbg()` 没有权限检查，响应中的 phpinfo 表格包含 `spx.http_key`。提取该 16 字符值后，可以访问：

```text
/?SPX_UI_URI=/data/reports/metadata&SPX_KEY=<key>
```

该接口返回 profiling report metadata，其中包含原始请求 URI 和 `recorded_call_count`。每次探测在 query string 中加入随机 `ident`，随后按 `http_request_uri` 搜索该 ident，避免把并发请求或前一次结果错配给当前候选。

### 2. 先恢复密码长度

比较函数先做严格长度检查：

```php
if (strlen($pw) != strlen($hp)) return false;
for ($i = 0; $i < strlen($pw); $i++) {
    if (!check_char($pw[$i], $hp[$i])) return false;
}
```

依次提交长度 0 到 63 的 `'0' * n`。长度错误时循环完全不进入；只有正确长度会至少调用一次 `check_char`，因此 metadata 中的调用总数出现明显最大值。源码生成密码时固定 `head -c 32`，实测侧信道也应恢复出 32；源码常量只能用于校验，不能替代攻击过程。

### 3. 用调用数逐字符恢复前缀

密码字符集为 `A-Za-z0-9`。设已知前缀为 $P$，测试候选字符 $c$ 时提交：

```text
P || c || "0" * (password_length - len(P) - 1)
```

所有探测长度相同，排除了长度分支。若 $c$ 错误，循环在该位置返回；若正确，会继续比较至少下一个填充字节，因此多发生一次 `check_char` 及相关调用。对 62 个候选读取 `recorded_call_count`，选择唯一较大的值并扩展 $P$。

最后一位没有“下一次比较”可用来制造增量：正确值会直接完成登录。参考 solver 因此恢复前 31 位后，对最后一位发送真实登录请求，直到响应出现 flag。

源码实例结果为：

```text
gctf{Y0u_C4n_Als0_S!d3Ch4nN3l_PhP_O_o_axNGgpno5ycGw85}
```

## 方法总结

本题不是测网络耗时，而是通过 profiler 读取确定性的函数调用数量，信号比传统 timing attack 更稳定。漏洞链为“未授权 phpinfo → SPX key 泄漏 → profiling metadata 越权读取 → 短路比较调用数侧信道 → 管理员登录”。修复应移除生产环境的 phpinfo 和公开 profiler UI，用 `hash_equals` 等常量时间比较，并确保调试/性能数据不会跨用户泄漏请求细节。
