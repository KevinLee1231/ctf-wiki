# hdHessian

## 题目简述

原始合并题解将 hdHessian 描述为 ezHessian 的加固版：较新 Hessian 在反序列化阶段黑名单限制了 `Runtime`、`ProcessImpl` 等直接执行类，额外 `java` WAF 使常见 payload 不可直接使用。预期目标是 Jetty Filter 内存马，而非一次性 DNS RCE；题目仍以 Hessian HTTP 输入为入口，归 `web`。

## 解题过程

### 两阶段加载思路

题解指出：直接把 class bytecode 写进 Hessian payload 时，其常量池含 `java`，无法绕过 WAF；而 Hessian 的黑名单关注反序列化可见类，不能等价阻止类加载期逻辑。可利用 `SwingLazyValue` 调用允许的静态方法完成两阶段动作：

1. 通过 `com.sun.org.apache.xml.internal.security.utils.JavaUtils.writeBytesToFilename` 将 Base64 化的恶意 XSLT 写到临时文件；
2. 通过 `com.sun.org.apache.xalan.internal.xslt.Process._main` 以 `-XT -XSL <file>` 处理 XSLT，由 XSLT 解码并加载字节码。

题解示例把 `java` 拆为 XML HTML 实体，如 `ja&#x76;a`，使 WAF 输入不含连续字符串，XML 解析后仍恢复正确 Java 扩展名。生成器继续用 overlong Hessian 输出编码对象图。

### Jetty Filter 内存马与验证

恶意类在加载时枚举线程上下文中的 Jetty `WebAppClassLoader`/`HttpConnection`，定位 servlet context，向 `_filterMappings` 注入 Filter。Filter 使用请求 header 作为命令并将输出写到响应中，提供稳定回显；这正是题解把它与 ezHessian 的 DNS-only RCE 区分开的原因。

原题解还明确说明实际赛中多为非预期解（时间盲注或 socket fd 回显），因此不能把这些视为已验证的正式链。当前材料没有 hdHessian 服务源码、Hessian 版本或完整可复现工程；本文只保留原题解已给出的两阶段路径和限制。

## 方法总结

- 核心技巧：把被 WAF 拦截的 class bytecode 移出 Hessian 字符串，借允许类写入并由 XSLT 二阶段加载，再安装 Jetty 内存马。
- 识别信号：Hessian 黑名单阻断直接 `Runtime`，但仍可调用类加载/文件写入相关 API；同时 WAF 按原始文本搜索 `java`。
- 复用要点：内存马持久化不等于可靠回显，必须分别验证加载、注册和请求命中；防御端应禁用不可信 Hessian 与危险 Xalan/反射路径。
