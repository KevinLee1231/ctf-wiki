# GreyCTF2024 Greycat's Adventure - Achievement 2 WP

## 题目简述

本题与同系列挑战共用一款 Unity IL2CPP 游戏，要求购买商店中的全部物品。预期动态做法是修改金币余额；静态做法则是从 IL2CPP 元数据中提取明文 flag，并根据“无限金币、无限力量”的语义确定对应项。

## 解题过程

先在游戏数据目录中搜索 flag 前缀：

```bash
grep -aR -n "grey{" "GreyCat's Adventure_Data"
```

命中位置为：

```text
GreyCat's Adventure_Data/il2cpp_data/Metadata/global-metadata.dat
```

从该文件筛出可打印字符串：

```bash
strings "GreyCat's Adventure_Data/il2cpp_data/Metadata/global-metadata.dat" \
  | grep -F "grey{"
```

其中与商店和金币机制直接对应的是：

```text
grey{unl1m1t3d_m0n3y_unl1m1t3d_p0w3r_kl2j1fd}
```

动态复现时，先在商店界面记下余额，用内存扫描器搜索该数值；购买一次物品后以新余额继续筛选，定位余额变量。把它改成足够大的正整数，再购买所有商品，即可触发成就。静态提取与动态触发得到的语义一致。

## 方法总结

处理共享附件的多题组时，不能把所有答案混写成一篇。应先枚举元数据中的候选字符串，再根据每道子题的触发条件逐一映射。本题的决定性线索是商店余额，对应 flag 中的 `unl1m1t3d_m0n3y`。
