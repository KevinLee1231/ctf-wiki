# ezjava

## 题目简述

题目模拟“注册中心同步服务端配置”的场景：registry 侧保存 Java 原生反序列化黑名单，server 侧定时同步。攻击链分两段：先利用 registry 侧 Sofa Hessian 反序列化，在黑名单绕过条件下触发 getter/toString 到 RCE；再控制 registry 返回给 server 的黑名单内容，关闭 server 侧 JEP 290 防护并利用 Fastjson/`TemplatesImpl` 类链拿到 server 侧 flag。

## 解题过程

题目模拟真实动态配置中心的架构。registry 侧用于存放相关配置，server 侧会定时同步这些配置；这里的配置指 Java 原生反序列化黑名单。

### 注册中心侧 Hessian 反序列化漏洞

从给出的代码可以看到，registry 侧存在 Hessian 反序列化漏洞。不过这里使用的是 Sofa Hessian，支持加载 Hessian 黑名单来拦截常见 Hessian 利用链。

`/hessian/deserialize` 接口会对 `base64Str` 做 Base64 解码后交给 Hessian 反序列化，成功时返回 `deserialize success`。

黑名单从 `resources/security/hessian_blacklist.txt` 加载。仔细检查后可以发现，它拦截了大多数常见利用链，但没有拦截当时较常见的 Fastjson 利用链；依赖中也能找到对应的 Fastjson 依赖。

因此当前主要问题是：在不能出网的条件下找到可用 getter，并触发 `toString()` 调用。

### 不出网触发 Getter

外部参考中的 `ysomap` Hessian payload 说明了这里需要的关键构造：把攻击对象塞进 `CannotProceedException.resolvedObj`，再用 `ContinuationDirContext` 的 getter 触发 `NamingManager.getContext`，最终进入 JNDI/BeanFactory 相关对象实例化流程。这里不依赖网络加载，而是借助本地已有类和 getter 调用链触发后续执行。

```
CannotProceedException cpe = new CannotProceedException();
ReflectionHelper.setFieldValue(cpe, "cause", null);
ReflectionHelper.setFieldValue(cpe, "stackTrace", null);
cpe.setResolvedObj(obj);
ReflectionHelper.setFieldValue(cpe, "suppressedExceptions", null);
Object ctx = ReflectionHelper.newInstance(
        "javax.naming.spi.ContinuationDirContext",
        new Class[]{CannotProceedException.class, Hashtable.class},
        cpe, new Hashtable<>());
```

`ContinuationDirContext` 有多个 getter 可以触发 `getTargetContext` 函数。

`getTargetContext()` 会通过 `NamingManager.getContext(...)` 解析目标上下文，为后续 JNDI/命名服务相关调用提供入口。

要借助类属性 `cpe` 中的 `resolvedObj` 对象触发 `NamingManager.getContext`，需要理解 JNDI 的基本原理。

熟悉 JNDI 机制的话会知道，如果当前传入的 `obj` 是 reference 对象，就可以触发任意 `BeanFactory` 的 `getObjectInstance` 函数。随后就可以考虑使用 Tomcat 的 Expression Language（EL）执行任意代码。相关利用方式有很多，这里不再展开。

`TomcatRefBullet.java` 这一类利用的核心是构造 Tomcat BeanFactory 可解析的 Reference，使 EL 表达式或等价 gadget 在本地类路径中执行。实际利用时先准备可被目标加载的恶意 JAR 或本地类，再让上面的 getter 链触发它。

### 触发 toString 函数

参考 Dubbo 中通过构造畸形序列化 payload 触发 `toString` 调用的方法，这里以

`com.caucho.hessian.io.AbstractMapDeserializer` .
为例。

自定义 `AbstractDeserializer` 的 `readObject` 中会调用 getter/setter 路径，可用来触发目标类方法。

构造一个偏离正常格式的序列化 payload 会触发异常。由于解析过程是递归的，底层 `obj` 会先被解析，因此可以在抛出异常的位置直接触发当前 `obj` 的 `toString` 函数。

把这两部分组合起来，就可以在不出网的情况下实现任意代码执行。

### 服务端侧 Java 原生反序列化漏洞

由于 flag 位于 server 侧，必须进一步打到 server 才能获取 flag。server 侧只有一个名为 `status` 的接口，用于更新 server 侧的反序列化黑名单，也就是利用 JEP 290 的过滤机制。

最终 gadget 通过构造 `ReadObject` / `MethodClass` 一类对象，在 Hessian 反序列化过程中触发方法调用链。

该接口的流程是先执行反序列化，再更新当前黑名单。因此如果能控制 registry 上 `blacklist/jdk/get` 端点返回的内容，就可以触发漏洞。

前面已经在 registry 上获得了任意代码执行能力，因此可以尝试用 webshell 覆盖当前接口响应，进而操控 registry 行为。这类 webshell 覆盖接口响应的要点是：在 Java Web 容器中挂入 filter/servlet/agent 逻辑，拦截指定 URL 的响应并返回攻击者控制的内容。这里用它把 registry 的黑名单接口改成返回空 `denyClasses`，从而让 server 认为没有需要拦截的危险类。

首先，用 webshell 覆盖 registry 上的 `blacklist/jdk/get` 接口，让它返回空列表替代原本的 `denyClasses`。这一步等价于移除了 server 侧的安全防护。

移除 Java 原生反序列化防护后，就可以利用 server 侧依赖。依赖中存在 Fastjson，因此可以直接考虑使用 `fastjson` 与 `templateImpl` 组合实现任意代码执行。

随后，由于 server 不能出网，只能覆盖 server 侧 `status` 接口，让它返回我们需要的内容。方法与前面类似，也是用 webshell 覆盖接口响应并取回 flag。由于 server 不能直接出网，利用链的重点不是回连，而是把最终信息写入/返回到一个已有可访问接口。

最后，从 `/client/status` 接口读取 flag 内容。

## 方法总结

- 核心技巧：Hessian 反序列化黑名单绕过、getter/toString 触发链、JEP 290 黑名单同步逻辑劫持，以及 server 侧 Fastjson/`TemplatesImpl` RCE。
- 识别信号：出现“注册中心同步安全配置”“先反序列化再刷新黑名单”“目标不能出网但能访问本地接口”时，应优先考虑控制配置源或接口响应。
- 复用要点：外部参考不能只看 payload 名称，必须提取其中的调用机制。本题真正关键是 `CannotProceedException` + `ContinuationDirContext` 如何把普通对象推进 JNDI getter 链。
