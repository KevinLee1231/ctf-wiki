# hintbot

## 题目简述

WebSocket 接口在 debug=true 时返回输入到所有提示键中最近一项的 Levenshtein 距离。flag 被转换为 maple words 格式后也加入提示键字典，因此这个距离值构成逐字符 oracle，可以从末尾向前恢复未知字符串。

## 解题过程

服务先把 maple{a_b} 转成 maple a b。选择一个足够长的候选 maple 加若干占位符，从最后一个位置开始枚举字符集。对每个候选发送：

~~~json
{"msg":"候选字符串","debug":true}
~~~

记录响应 dist，保留距离最小的字符；若多个字符并列则分支继续。逐位置向前递归，并对可能长度从 60 到 21 试探。当候选进入阈值 2 以内时，响应不再包含 dist 而返回 looks like you're winning，可确认完整文本：

~~~text
maple i got robert on my team
~~~

按题面格式还原为：

~~~text
maple{i_got_robert_on_my_team}
~~~

## 方法总结

相似度、编辑距离和置信度都可能泄漏秘密的局部正确程度。恢复时不应贪心丢弃并列最优字符，而要保留分支；同时把长度也纳入搜索。生产接口不应向未授权用户返回调试距离，比较秘密时只应给出固定、无梯度的结果。
