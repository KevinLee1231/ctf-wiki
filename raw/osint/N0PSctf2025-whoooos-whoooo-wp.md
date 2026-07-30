# Whoooo's whoooo

## 题目简述

绑架案链的最后一题要求找出幕后首领的真实身份。需要从会话中暴露的预约表单继续追踪表单作者、TopiaNews 资料页和 Tor 隐藏站点。

## 解题过程

会话 ID `1337` 中包含一个 Framaforms 预约链接。表单本身没有直接显示首领姓名，但配置存在两处信息泄露：

1. 结果页面被设置为公开，可以确认它确实用于绑匪之间安排会面；
2. “联系作者”功能暴露了创建者使用的昵称 **Digitowl**。

以 `Digitowl` 回到 TopiaNews 搜索人物资料，可以找到其公开档案。资料页没有写真实姓名，却留下了一个 Tor v3 地址：

```text
m7o263b6fetopgij5bombfnpxvapday6cj4jqmsi4cdxp3axb6arjtid.onion
```

使用 Tor Browser 访问该地址后，隐藏站点明确把组织首领称为：

```text
Elias Nightshade
```

这一步提供的是页面中的明确身份说明，而不是根据昵称 `Digitowl` 猜测。按题目格式转为小写并用下划线连接：

```text
N0PS{elias_nightshade}
```

## 方法总结

完整 pivot 为“历史会话 → Framaforms 表单 → 作者昵称 Digitowl → TopiaNews 人物页 → onion 地址 → 真实姓名”。表单的公开结果证明用途，“联系作者”功能提供跨站关联标识，Tor 页面才是最终身份的直接证据。正文已经记录 onion 地址和页面结论，即使外部表单日后下线，也能理解并复核整条推理链。
