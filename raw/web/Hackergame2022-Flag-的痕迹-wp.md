# Flag 的痕迹

## 题目简述

目标站点是 DokuWiki 2022-07-31a。管理员曾把 flag 提交到首页，随后删除内容，并在配置中关闭“最近修改”和“历史版本”操作。DokuWiki 不使用数据库，而是把页面和修订记录保存在文件系统中；关闭前端 action 只隐藏入口，并未删除旧版本。配置还遗漏了 `diff` 操作，因此可以从差异页面读取历史内容。

## 解题过程

题目源码中的关键配置为：

```php
$conf['disableactions'] =
    'backlink,index,recent,revisions,search,register,resendpwd,' .
    'profile,profile_delete,edit,check,rss,subscribe,unsubscribe,' .
    'source,export_raw';
```

可以看到 `recent` 和 `revisions` 被禁用，但列表里没有 `diff`。DokuWiki 的 action 由 `do` 查询参数选择，所以在首页 URL 上请求：

```text
/doku.php?id=start&do=diff
```

差异页面仍会读取该页在磁盘上的修订索引，并提供旧版本选择框。选择删除 flag 之前的版本，与当前版本比较，就能在删除内容一侧看到完整 flag。

如果界面没有直接显示历史入口，也可以从页面表单或开发者工具中确认 `do=diff` 请求；关键是直接访问遗漏的 action，而不是枚举每一秒的修订时间戳。

关闭功能后旧版本仍存在，是因为 DokuWiki 的修订数据没有被清理。禁用 action 只属于访问控制或界面配置，不能替代数据删除。

## 方法总结

- 核心技巧：枚举 DokuWiki 的 action，利用未禁用的 `diff` 读取已经隐藏的历史修订。
- 识别信号：应用声称“关闭历史记录”，但底层使用文件型版本存储，配置仅列出部分被禁用操作。
- 复用要点：检查同一资源的所有功能入口，而不只依赖导航菜单；真正清除敏感历史需要删除或重写底层修订数据，并验证旁路 API/action 已同步受控。
