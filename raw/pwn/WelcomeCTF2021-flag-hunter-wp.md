# Flag Hunter

## 题目简述

WelcomeCTF2021 的 Flag Hunter 把漏洞包装成回合制游戏。Flag Guardian 的生命值使用有符号 `char` 保存，取值范围只有 $-128\ldots127$；当生命值超过 127 时会绕回负数，而胜利条件正是 `guardian.health <= 0`。

## 解题过程

源码中的类型差异是关键：

```c
struct Guardian {
    int type;
    char health;
    int damage;
    int heal;
};
```

选择 Mage，并重新连接直到出现生命值 80、伤害 10、每回合回复 4 的 Guardian。另一种 Guardian 伤害 30，玩家无法用相同节奏存活。

Mage 初始有 42 生命和 50 法力。循环执行以下三步：

1. 使用 Mana Shield，消耗 25 法力，自己不受伤，Guardian 回复 4；
2. 再使用一次 Mana Shield，法力降为 0，Guardian 再回复 4；
3. 使用 Magic Book 把法力恢复到 50，自己承受 10 点伤害，Guardian 再回复 4。

每轮三步让 Guardian 增加 12 点生命，玩家只损失 10 点。状态从 80 依次增长，最终由 124 加 4 变成 128；写入有符号 8 位 `char` 后表示为 -128。循环末尾检查：

```c
if (guardian.health <= 0) {
    puts("You win");
    puts("greyhats{1nt3rger_OooOooverflow_in_3ss3nce}");
}
```

因此无需击打 Guardian，反而要让它持续治疗自己直至整数溢出。

## 方法总结

本题的漏洞是业务逻辑中的窄整数溢出。源代码把玩家生命值设为 `int`，却把 Guardian 生命值设为 `char`，且没有上界检查。审计游戏或计数器逻辑时，应同时检查变量宽度、符号性以及加减法发生后的判定顺序。
