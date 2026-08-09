# jane_doe

## 题目简述

题目从嫌疑人 `A. M. March` 的用户名规律出发，串联 Twitter、Instagram 和 Wayback Machine。flag 被拆成两半，并分别使用 ROT13 与 Base64 包装。

## 解题过程

根据“prefers underscores”先找到 Twitter 用户 `A_M_March`。简介中的 `a00om{abg_fb_snfg_yznb}` 经 ROT13 只得到诱饵 `n00bz{not_so_fast_lmao}`。由账号姓名 April May March 继续定位 Instagram 用户 `april_m_march`，其帖子说明中两段 Base64 解码为叙事文字，尾段 ROT13 得到后半段：

```text
k1ll3d_j4n3_d03!}
```

Twitter 中“删除敏感信息”的提示指向网页存档。查看该账号的历史快照，删除推文内的 Base64 解码为提示文字，ROT13 部分给出前半段：

```text
n00bz{1t_w45_jA_Jun3_wh0_
```

拼接得到：

```text
n00bz{1t_w45_jA_Jun3_wh0_k1ll3d_j4n3_d03!}
```

## 方法总结

OSINT 链中的假 flag 不能仅凭格式接受。应验证账号之间的头像、姓名和叙事关联，并把当前页面与历史快照视为两类独立证据；每次 Base64 或 ROT13 解码后也要判断其在整个故事中的作用。
