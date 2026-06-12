# real_cloud_storage

## 题目简述

本题模拟云对象存储 SDK 被业务方错误使用的场景。后端把 OSS/NOS SDK 所需的 `endpoint`、`bucket`、`key`、`file` 等参数都交给前端控制，导致客户端请求目标可控，形成有限 SSRF。

题目附件是仿云存储/云函数环境，前端可控 OSS/NOS SDK 的 endpoint、bucket、key 和上传内容。单纯 SSRF 无回显，进一步审计 `nos-sdk-java/1.2.2` 可发现其处理异常响应 XML 时没有禁用外部实体，因此可通过恶意 OSS 错误响应把 SSRF 转成 XXE。

## 解题过程

题目源码见 [`Li4n0/My-CTF-Challenges/D^3CTF2021_real_cloud`](https://github.com/Li4n0/My-CTF-Challenges/tree/master/D%5E3CTF2021_real_cloud)。下文保留了源码中的可控 OSS/NOS 参数、SDK 版本识别、SSRF 到异常 XML 解析的 XXE 利用链。

题目表面是后端代用户上传文件到 OSS/NOS，直觉上容易先看云存储服务端鉴权。但这里更关键的是 SDK 使用方式：业务把 `OSSClient` 所需的 `endpoint`、`bucket`、`key` 等连接参数交给前端传入，这会让 SDK 主动向攻击者指定的地址发起请求。

例如把 `endpoint` 和 `bucket` 指向攻击者服务器：

```json
{
  "endpoint": "target.com",
  "key": "index.php",
  "bucket": "www",
  "file": "MTIzMTIzMTIzMQ=="
}
```

攻击者服务器可以收到来自 SDK 的 `PUT /index.php` 请求，`User-Agent` 中能看到版本为 `nos-sdk-java/1.2.2`。这一步只能得到无回显 SSRF，本身还不能直接读文件，因此需要继续审计 SDK 对服务端响应的处理逻辑。

根据 `User-Agent` 定位到网易云对象存储的 [`nos-java-sdk`](https://github.com/NetEase-Object-Storage/nos-java-sdk)。由于当前操作是上传文件，重点看 `putObject` 失败时的异常处理路径。容易忽略的是：虽然请求参数由用户控制，服务端返回包同样由攻击者控制。NOS/S3 风格协议在错误响应中使用 XML，若 SDK 直接用默认 XML 解析器处理错误 body，就可能把可控响应转成 XXE。类似对象存储 SDK 的异常 XML 解析问题也可参考 IBM COS SDK 的相关讨论：<https://github.com/IBM/ibm-cos-sdk-java/issues/20>。

`nos-sdk-java/1.2.2` 的 `NosErrorResponseHandler` 会在错误响应存在 body 时直接调用 `XpathUtils.documentFrom(errorResponse.getContent())`，然后从 XML 中取 `Error/Message`、`Error/Code`、`Error/RequestId` 和 `Error/Resource`，没有看到禁用外部实体的处理。关键逻辑如下：

```java
public ServiceException handle(HttpResponse errorResponse) throws Exception {
    if (errorResponse.getContent() == null) {
        String requestId = errorResponse.getHeaders().get(Headers.REQUEST_ID);
        NOSException ase = new NOSException(errorResponse.getStatusText());
        ase.setStatusCode(errorResponse.getStatusCode());
        ase.setRequestId(requestId);
        fillInErrorType(ase, errorResponse);
        return ase;
    }

    Document document = XpathUtils.documentFrom(errorResponse.getContent());
    String message = XpathUtils.asString("Error/Message", document);
    String errorCode = XpathUtils.asString("Error/Code", document);
    String requestId = XpathUtils.asString("Error/RequestId", document);
    String resource = XpathUtils.asString("Error/Resource", document);

    NOSException ase = new NOSException(message);
    ase.setStatusCode(errorResponse.getStatusCode());
    ase.setErrorCode(errorCode);
    ase.setRequestId(requestId);
    ase.setResource(resource);
    fillInErrorType(ase, errorResponse);
    return ase;
}
```

因此最终利用方式是把 `endpoint` 指向攻击者域名，在攻击者服务器上返回异常状态码和带外实体的 XML payload。SDK 会把这个错误响应交给 XML 解析器，原本无回显的 SSRF 就变成了可读取本地文件或内网资源的 XXE。

## 方法总结

- 核心技巧：云 SDK 参数可控导致 SSRF，再利用 SDK 对服务端错误 XML 的不安全解析触发 XXE。
- 识别信号：业务层把 `endpoint`、`bucket`、`key` 这类云 SDK 连接参数交给用户；请求无回显但攻击者可以控制服务端响应。
- 复用要点：审云服务题时不要只攻击云服务端鉴权，也要审 SDK 客户端如何处理异常响应、重定向、XML/SOAP 解析等“返回包可控”的路径。

