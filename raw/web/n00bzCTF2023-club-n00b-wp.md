# Club_N00b

## 题目简述

主页按钮把用户带到 `/check?secret_phrase=nope`，而页面正文突出显示单词 `radical`。后端只对查询参数做固定字符串比较。

## 解题过程

把 URL 改为：

```text
/check?secret_phrase=radical
```

Rust 后端在匹配到小写 `radical` 后返回临时重定向，并把环境变量中的 flag 放入目标页查询参数：

```rust
"radical" => HttpResponse::Ok()
    .status(StatusCode::TEMPORARY_REDIRECT)
    .append_header(("Location", format!("/members_only.html?flag={}", flag)))
```

浏览器跟随重定向后显示：

```text
n00bz{see_you_in_the_club_acting_real_nice}
```

## 方法总结

这是一道参数逻辑题，决定性线索就在主页可见文本中。遇到按钮生成的固定查询参数，应先修改其值并结合后端分支验证，而不是把普通重定向误判为复杂漏洞。
