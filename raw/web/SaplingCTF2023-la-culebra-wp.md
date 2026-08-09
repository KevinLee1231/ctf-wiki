# la-culebra

## 题目简述

用户可写入 Redis 中的蛇知识，读取时服务对值 Base64 解码后直接交给 render_template_string，形成 Jinja2 SSTI。flag 存在 Redis 的 flag 键中；路由禁止直接读取该键，且输出以 maple{ 开头时会重定向。需要通过 SSTI 执行 Python，把 flag 去掉首字符后重新编码到另一个键。

## 解题过程

向 /newsnakefact 写入 Jinja 表达式。通过 object 子类找到可执行子进程的类，启动 Python 连接同一 Redis，读取原始 flag，然后写入新键 ok：

~~~python
import redis, base64
r = redis.Redis(host="localhost", port=6379)
flag = r.get("flag")
r.set("ok", base64.b64encode(flag[1:]))
~~~

上述代码可放入 subprocess 的 python3 -c 参数，由 SSTI 触发。部署拓扑若把 Redis 暴露为服务名，则主机名改为 redis；应以实际 compose 或 Pod 网络为准。

随后请求 /getsnakefact/ok。服务 Base64 解码后得到 aple{...}，不再命中 startswith("maple{")，页面返回主体。手工补回开头 m：

~~~text
maple{tH3res_4_snek_In_My_keY_valu3_dict!!}
~~~

## 方法总结

SSTI 的价值不只在读取当前容器文件，还可访问应用已授权的后端资源。绕过这里的关键是理解 Redis 中 flag 键保存原文，而普通事实键保存 Base64；复制时去掉一个字符，既满足存储格式又避开前缀过滤。根本修复是禁止对用户内容调用 render_template_string。
