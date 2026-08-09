# MCA

## 题目简述

/secret 同时支持 GET 和 POST。GET 返回的伪 404 页面注释中藏有 Base64 字符串，POST 则直接返回另一段 Base64。将 GET 中真正的前缀与 POST 结果拼接即可得到 flag；这不是密码攻击，而是 HTTP 方法处理失当造成的信息泄漏。

## 解题过程

先请求：

~~~http
GET /secret HTTP/1.1
Host: target
~~~

HTML 注释中的字符串解码为：

~~~text
maple{y3s_I_kn0w_ ...wait where's the other half?\x05
~~~

空格后的句子是提示，不属于 flag；保留到第一个空格前的 maple{y3s_I_kn0w_。再对同一路径发 POST：

~~~http
POST /secret HTTP/1.1
Host: target
Content-Length: 0
~~~

响应 Base64 解码为：

~~~text
_th4t_r3stful_4PIs_dont_w0rk_lyk_d1s}
~~~

两段边界各自都有一个下划线，必须全部保留：

~~~text
maple{y3s_I_kn0w__th4t_r3stful_4PIs_dont_w0rk_lyk_d1s}
~~~

## 方法总结

侦察 Web 路由时应检查 GET、POST、HEAD、OPTIONS 等实际允许方法，不要只相信页面显示的 404 文案。Base64 只是表示层编码，解码后还要识别提示文本、控制字节和真实字段边界。API 应让不同方法具有清晰一致的授权策略，不能把秘密分散在可公开调用的分支中。
