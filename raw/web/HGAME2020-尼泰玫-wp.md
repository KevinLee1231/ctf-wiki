# 🐔尼泰玫

## 题目简述

题目是浏览器中的 JavaScript 篮球小游戏，总分超过 30000 后才会向后端请求 flag。页面脚本 `js/game.js` 同时包含分数变量、请求接口的混淆拼接方式，以及以当前 Unix 时间戳计算的 MD5 校验值。

## 解题过程

不需要实际刷分。打开开发者工具检查 `js/game.js`，可以看到提交逻辑把分数与当前秒级时间戳的 MD5 拼成：

```text
<score>|md5(<unix-seconds>)
```

接口地址和字段名由页面已有变量拼接并经 Base64 解码。为了避免重新还原所有局部变量，可在游戏页面的 Console 中复用原执行环境，只把分数改为足够大的值：

```javascript
var score = 300001;

(function () {
    const encodedUrl = (me + "4ay5oZ") + rt + (rou + "1pdA");
    const encodedKey = k + "cmU=";
    const timestamp = Date.parse(new Date()) / 1000;

    $.post(
        atob(encodedUrl),
        atob(encodedKey) + "=" + score + "|" + md5(timestamp),
        function (data) {
            alert(data);
        }
    );
})();
```

这段代码依赖页面已经初始化的 `me`、`rt`、`rou` 和 `k`，因此应在原游戏页面执行，而不是当成独立 Node.js 脚本运行。另一种等价做法是在球结算前修改全局分数对象，使原有提交函数自行发送请求。

[公开参赛者记录](https://asjet.dev/2020/02/week1/)给出的验证结果为：

```text
hgame{j4vASc1pt_w1ll_tel1_y0u_someth1n9_u5efu1?!}
```

## 方法总结

- 核心技巧：阅读前端脚本，直接复用客户端的接口拼接和签名逻辑提交伪造高分。
- 识别信号：胜利条件、分数和网络请求全部位于可控 JavaScript 中，后端又接受客户端提供的分数。
- 复用要点：客户端变量和时间戳签名不构成可信校验；服务端必须独立维护游戏状态，不能只验证前端可重算的数据。
