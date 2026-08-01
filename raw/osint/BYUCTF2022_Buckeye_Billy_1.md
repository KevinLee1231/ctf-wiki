# BYUCTF 2022 - Buckeye Billy #1

## 题目简述

Buckeye Billy 的生日题给出三个自定义 Wordle 链接，并用 “sweet tooth”“history loving”提示最终答案是一家与甜食和历史地址有关的商店。

## 解题过程

分别解开三个 Wordle，得到：

```text
water
probe
calls
```

把它们按 `what3words` 格式组成 `water.probe.calls`。同名组合在不同语境可能产生歧义，但 “Buckeye State” 指向 Ohio，正确坐标落在 Cincinnati 的：

```text
967 E McMillan St, Cincinnati, Ohio
```

当前位置并没有题面所说的礼物店，因此继续使用 “history” 提示查地址历史。Graeter's 的[官方历史页面](https://www.graeters.com/about-us/our-history)说明：Louis Graeter 与妻子 Regina 在 1900 年迁到 967 E McMillan Street，并在店面前制作、售卖冰淇淋。它同时满足 Ohio、历史地址和 “sweet tooth” 三个条件。

商店名按小写提交：

```text
byuctf{graeters}
```

## 方法总结

what3words 给出的只是坐标，不保证当前街景直接出现答案。题面中的州别、嗜甜和历史分别用于消歧、确定业态、追溯旧址；将三类证据交叉后，Graeter's 才是唯一合理结果。
