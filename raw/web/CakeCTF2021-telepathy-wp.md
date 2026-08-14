# CakeCTF2021 telepathy

## 题目简述

后端 Go Echo 在根路径直接返回 `flag.txt`，前面的 OpenResty 则用 Lua body filter 搜索 `\w*\{.*\}`，把看起来像 flag 的响应体替换成提示语。题目没有要求绕过文件权限；关键在于 HTTP Range 会让后端先产生局部响应，而过滤器只能看到这个局部正文。

## 解题过程

### 比较应用层和代理层的处理顺序

后端配置只有一条文件路由：

```go
e.File("/", "public/flag.txt")
```

OpenResty 的过滤逻辑为：

```lua
ngx.arg[1] = ngx.re.gsub(
    ngx.arg[1],
    "\\w*\\{.*\\}",
    "I'm sending the flag to you by telepathy... Got it?\n"
)
```

正常请求返回完整 `CakeCTF{...}`，正则能同时看到左花括号和右花括号，于是整段被替换。HTTP `Range` 请求则由静态文件响应逻辑切出一部分正文，再交给代理过滤；只要分片不包含完整 `{...}`，正则就无法命中。

### 请求不含左花括号的后半段

固定前缀 `CakeCTF{` 长 8 字节。直接请求从偏移 8 到文件结尾的范围：

```sh
curl -sS -H 'Range: bytes=8-' http://target/
```

返回体只包含花括号内内容及末尾 `}`，没有左花括号，因此不匹配过滤正则。把已知前缀补回即可：

```sh
body=$(curl -sS -H 'Range: bytes=8-' http://target/)
printf 'CakeCTF{%s\n' "$body"
```

得到：

```text
CakeCTF{r4ng3-0r4ng3-r4ng3}
```

更通用的做法是发出多个互不包含完整模式的 Range 请求，再按字节偏移拼接；本题因前缀已知，一次后半段请求就足够。

## 方法总结

- 响应过滤器看到的未必是源文件整体；Range、分块传输和缓存层都可能改变它实际处理的数据单元。
- 黑名单正则只有在完整敏感串位于同一过滤缓冲区时才有效，不能替代后端授权控制。
- 题目虽放在 Misc，但决定性障碍是 HTTP Range 与反向代理响应过滤的组合，因此归入 Web 更准确。
