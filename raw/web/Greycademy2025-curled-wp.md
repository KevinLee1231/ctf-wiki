# curled

## 题目简述

网页按钮会向 `/flag` 提交普通 POST，但服务端用不同 HTTP 方法逐步给出下一条线索。按照响应提示依次发送 `POST`、自定义 `POAST` 和 `HEAD`，最终 flag 出现在响应头而不是正文。

## 解题过程

首页表单提交：

```http
POST /flag
Content-Type: application/x-www-form-urlencoded

poast=flag
```

服务端返回 405，并明确提示改发 `POAST`。使用 curl 保留响应头观察每一步：

```bash
curl -i -X POST "$BASE/flag" -d 'poast=flag'
curl -i -X POAST "$BASE/flag" -d 'poast=flag'
```

第二个响应给出 token `sesquipedalian`，并提示对 `/flag` 使用 `HEAD`。`do_HEAD` 只检查 `Authorization` 头中是否包含该字符串：

```bash
curl -sSI "$BASE/flag" \
  -H 'Authorization: sesquipedalian'
```

返回头包含：

```http
HTTP/1.0 200 OK
Server: greycademy
Flag: grey{curl_th3_fl4g!}
```

由于 HEAD 响应按规范没有可见正文，只看浏览器页面会错过结果，必须检查响应头。

## 方法总结

HTTP 不只有 GET 和 POST，自定义方法也能被服务器路由。排查协议谜题时应保存状态码、响应头和正文，认真跟随服务端提示；curl 的 `-X` 可指定任意方法，`-I` 或 `-sSI` 则用于发送 HEAD 并显示头部。
