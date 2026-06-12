# trustSQL-plus WP

## 题目简述

服务端接收上传的 SQLite 数据库，然后以低权限运行 `/chall/sqlite3 /db.sqlite "SELECT * FROM vuln;"`。题目是 2025 QWB Finals trustSQL 的加强版，移除了原题中依赖的 `PRAGMA trusted_schema=1`。

预期漏洞点在 SQLite FTS4 虚表的 `compress` / `uncompress` 选项。该问题后来已在 SQLite 上游修复，修复记录见 <https://sqlite.org/src/info/26f39ac806a5582f>。

## 解题过程

SQLite 在 `trusted_schema=0` 时会限制 schema 中触发 unsafe/direct-only 函数，例如 `load_extension`、`edit` 等，防止攻击者通过恶意 view/trigger 执行危险函数。但 FTS4 虚表被认为是 safe 的扩展，仍然可用。

问题在于 FTS4 的 `compress` 和 `uncompress` 参数允许用户指定函数名。SQLite 源码中 `fts3ReadExprList` 与 `fts3WriteExprList` 会把这个函数名拼进后续 SQL 片段，例如：

```c
fts3Appendf(pRc, &zRet, ",%s(x.'c%d%q')", zFunction, i, p->azColumn[i]);
fts3Appendf(pRc, &zRet, ",%s(?)", zFunction);
```

当之后查询 FTS4 虚表时，`fts3SqlStmt` 会把这些片段拼成完整 SQL 并重新 `sqlite3_prepare_v3`。这个执行上下文相当于新的顶层语句，从而绕过了 `trusted_schema=0` 对 unsafe 函数的限制。

因此可以创建带恶意 `compress` / `uncompress` 的 FTS4 虚表，让 SQLite 在查询 `vuln` 时调用本不该在 schema 中调用的函数。官方思路是先借此调用单参数函数 `sha1_query` 执行任意 SQL，再组合 `writefile` 和 `load_extension`：

1. 在上传的数据库中布置 `vuln` 和恶意 FTS4 虚表。
2. 通过 FTS4 `compress` / `uncompress` 触发 `sha1_query(...)`。
3. 用 `writefile` 写入自定义动态库。
4. 用 `load_extension` 加载动态库执行命令或读取 flag。

题目环境中 `/tmp` 不存在，因此不能直接走 `edit`。需要选择服务可写目录或数据库同目录写入扩展文件，再加载执行。flag 读取需要走挑战提供的读 flag 程序或等价命令执行路径。

## 方法总结

本题关键是绕过 `trusted_schema=0` 的上下文边界，而不是寻找普通 SQL 注入。FTS4 虚表的 `compress` / `uncompress` 把用户可控函数名拼入内部 SQL，并在后续查询时以顶层语句执行，使 unsafe 函数重新可达。拿到任意 SQL 后，再用 `writefile + load_extension` 完成 RCE。
