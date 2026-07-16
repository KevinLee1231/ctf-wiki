# week1Myrobots

## 题目简述

站点首页提示寻找 robots。服务器在 `/robots.txt` 中公开了一个未在页面链接中展示的文件名，直接访问该静态文件即可获得 flag。

## 解题过程

访问：

```text
/robots.txt
```

响应内容为：

```text
flag in FFFFl3gggg.txt
```

继续请求 `/FFFFl3gggg.txt`。容器构建脚本写入的实际内容是：

```text
0xGame{We1c0me!_Th1s_is_Your_Fl3g}
```

`robots.txt` 是给搜索引擎爬虫的访问建议文件，常用 `User-agent`、`Disallow` 等字段声明不希望抓取的路径；它不是访问控制机制，任何客户端仍可直接请求其中暴露的路径。本题正是利用这一点把它当作目录线索。

## 方法总结

- 核心方法：检查站点根目录的 `robots.txt`，按其中泄漏的文件名访问隐藏资源。
- 识别特征：首页出现 robot/crawler 提示，或常规目录中没有入口但 `robots.txt` 可访问。
- 注意事项：`robots.txt` 不能保护敏感文件；真实系统不应把密钥、备份或管理路径仅靠 `Disallow` 隐藏。
