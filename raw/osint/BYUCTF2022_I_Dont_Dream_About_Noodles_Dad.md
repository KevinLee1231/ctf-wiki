# BYUCTF 2022 - I Don't Dream About Noodles, Dad

## 题目简述

附件是《功夫熊猫》角色 Po 的雕像局部，问题询问谁的签名位于 Po 脚下。赛事背景和室内陈设提示雕像与 BYU 校园有关。

![BYU Talmage Building 内的 Po 雕像局部](./BYUCTF2022_I_Dont_Dream_About_Noodles_Dad/byu-po-statue.png)

## 解题过程

反向搜图或以 “Kung Fu Panda Po statue BYU” 检索，可定位到 Brigham Young University Talmage Building 内的 Po 雕像。继续查询 BYU 对该雕像和参与影片校友的报道，而不是只查角色配音演员。BYU 校报的 [5 campus locations you didn't know existed](https://universe.byu.edu/2012/09/27/5-campus-locations-you-didnt-know-existed/) 在 Po 雕像条目中明确给出 Jason Turner 这一关联。

校园报道把该雕像与参与《Kung Fu Panda》制作的 BYU 校友 Jason Turner 联系起来，并指出相关签名信息。题目问的是脚下签名者，按 `Firstname_Lastname` 格式提交：

```text
byuctf{Jason_Turner}
```

旧官方题解引用的另一篇文章主要谈到签名海报，不能单独证明“脚下签名”；结论应由明确提到 Talmage Building 的校园报道、雕像现场问题以及 Jason Turner 与影片制作的关系共同确认。

## 方法总结

识别角色只是第一步，环境与赛事主办方才是定位雕像的关键。地标确认后，应转向本地机构报道寻找“谁参与制作、谁留下签名”这类细节，并警惕二手题解引用与原文实际内容不完全一致。
