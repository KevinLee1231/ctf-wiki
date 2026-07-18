# Signin

## 题目简述

题目页面是一幅可交互涂色图，前端计算完成度并把结果作为 `score` 查询参数提交给当前页面。服务端没有重新计算实际涂色状态，而是直接依据客户端提供的数值决定是否返回 flag，因此不必手工完成涂色。

## 解题过程

阅读页面中的 `submitResult()` 可见核心请求：

```javascript
const score = parseFloat(
    document.getElementById('progressText').textContent
);

fetch(`/?score=${score}`)
    .then(response => response.text())
    .then(html => {
        if (score >= 100) {
            // 从返回 HTML 中提取 flag 并弹窗显示
        }
        window.history.pushState({score}, '', `/?score=${score}`);
    });
```

这里的涂色进度只存在于浏览器端，提交给服务端的却是一个可任意修改的普通 GET 参数。直接访问：

```text
/?score=100
```

即可走满分分支，响应正文中返回：

```text
RCTF{W3lc0m3_T0_RCTF_2025!!!}
```

总 PDF 中记录的比赛 IP 已属于临时服务入口，赛后没有复现价值，故不保留具体地址；路径和参数已经完整表达解法。

## 方法总结

- 关键问题是服务端信任客户端提交的 `score`，而不是前端显示逻辑本身。
- 遇到签到网页、小游戏或进度条时，应先查看请求参数和响应分支，再决定是否需要完成界面交互。
- 前端校验只能改善用户体验，不能证明状态真实；服务端若要防篡改，必须根据可信状态重新计算得分。
