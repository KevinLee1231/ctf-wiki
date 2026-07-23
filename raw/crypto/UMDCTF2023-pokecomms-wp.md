# Pokecomms

## 题目简述

题面提示 Pikachu “before he bytes you”，附件则由大量 `PIKA` 和 `CHU!` 组成。生成器把明文的每个字符按 8 位二进制展开，并采用：

```text
0 -> CHU!
1 -> PIKA
```

每个原始字节单独占一行，所以这是一个直接的二元表示层编码题。

## 解题过程

逐行按空白切分 token，将 `CHU!` 替换为 `0`、`PIKA` 替换为 `1`。每行应恰好恢复 8 位，再转换为一个 ASCII 字符：

```python
decoded = []

with open("pokecomms.txt", encoding="utf-8") as stream:
    for line in stream:
        bits = ""
        for token in line.split():
            if token == "CHU!":
                bits += "0"
            elif token == "PIKA":
                bits += "1"

        assert len(bits) == 8
        decoded.append(chr(int(bits, 2)))

print("".join(decoded))
```

`split()` 会去掉分隔空白，但不会破坏 `CHU!` 中的感叹号，因此不需要用字符串全局替换，也不会把行尾换行误当数据。

解码结果是一条很长的 Pikachu 故事，整体都位于同一个 flag 中：

```text
UMDCTF{P1K4CHU_Once_upon_a_time,_there_was_a_young_boy_named_Ash_who_dreamed_of_becoming_the_world's_greatest_Pokemon_trainer._He_set_out_on_a_journey_with_his_trusty_Pokemon_partner,_Pikachu,_a_cute_and_powerful_electric-type_Pokemon._As_Ash_and_Pikachu_traveled_through_the_regions,_they_encountered_many_challenges_and_made_many_friends._But_they_also_faced_their_fair_share_of_enemies,_including_the_notorious_Team_Rocket,_who_were_always_trying_to_steal_Pikachu._Despite_the_odds_stacked_against_them,_Ash_and_Pikachu_never_gave_up._They_trained_hard_and_battled_even_harder,_always_looking_for_ways_to_improve_their_skills_and_strengthen_their_bond._And_along_the_way,_they_learned_valuable_lessons_about_friendship,_determination,_and_the_power_of_believing_in_oneself._Eventually,_Ash_and_Pikachu's_hard_work_paid_off._They_defeated_powerful_opponents,_earned_badges_from_Gym_Leaders,_and_even_competed_in_the_prestigious_Pokemon_League_tournaments._But_no_matter_how_many_victories_they_achieved,_Ash_and_Pikachu_never_forgot_where_they_came_from_or_the_importance_of_their_friendship._In_the_end,_Ash_and_Pikachu_became_a_legendary_team,_admired_by_Pokemon_trainers_around_the_world._And_although_their_journey_may_have_had_its_ups_and_downs,_they_always_knew_that_as_long_as_they_had_each_other,_they_could_overcome_any_obstacle_that_stood_in_their_way}
```

## 方法总结

- 核心技巧：把两个固定 token 还原为二进制位，并利用一行对应一字节的边界直接解码。
- 识别信号：载体只由两种重复符号组成，且题面刻意提示 `bytes`，通常意味着二元映射和固定 8 位分组。
- 复用要点：先确认 token 映射方向和分组边界；若解出的首字节不是预期 flag 前缀，再尝试交换 0/1，而不必枚举复杂密码算法。
